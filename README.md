clj-probe
=========

clj-probe: systematic capture and processing of dynamic program state

## Introduction

Text-based logging infrastructure is an unfortunate historical
artifact. A log entry extracts subsets of information about dynamic
program state, but in a form that is difficult to automatically
process or relate back to the dynamic context in which it occurred.
Modern systems should seek to “late bind” the conversion of state from
a native internal representation to some serialized or human-readable
format and provide facilities for processing streams of state for
various purposes such as profiling, value analysis, 

[http://ianeslick.com/probe-a-library](Read the introductory post)

## Concepts

* Probe - Any mechanism that extracts dynamic state during program execution;
  a probe typically returns lexical, dynamic, and host information as well 
  as user-provided data.  Probes can include function entry/exit/exceptions 
  as well.  Easy to extend, for example, to add a watcher probe on an agent 
  or atom, etc.
* State - The map returned by a probe
* Policy - A name for one or more operations to transform probe state.  A singleton or seq of fns, fn symbols, and/or keywords naming 
  a policy.  Policy seqs operate much like ring middleware, but are easier to 
  modify at runtime to capture / ignore various subsets of state.
* Catalog - Global map storing named policies
* Config - maps namespace + tags to a policy, dynamically updatable
* Function probe - Default probes that can be added to any Var with a function value
* Log probe - Inject legacy library log statements into probe layer
* Sink - A function that side-effects some media (communications, storage

A policy function takes a single map, and returns an updated map.  Sinks return nil,
terminating the policy execution.

Reserved keys

- Static probes: :ts, :ns, :line, :thread-id, :tags
- Function probes: :fname, :fn, :args, :return, :exception

Reserved tags
- Standard log hierarchy: :trace, :debug, :info, :warn, :error
- Function probes: :fn, :exit-fn, :enter-fn, :except-fn

## Documentation by Example

    (use '[probe.core :as p])
	(use '[probe.sink :as sink])
	(use '[probe.policy :as policy])

Define a simple policy to print the raw state in a log-like format

	(p/set-policy! :console '[sink/console-log])

For probe points with :debug tags or above, handle them with the :console policy

    (p/set-config! 'user :debug :console)
    
    (p/probe #{:warn} :value 10)
	=> nil

    (p/probe #{:debug} :value 10)
    2013-05-10T22:59.212 user:1 {:thread-id 80 :value 10}
	=> nil

But we don't want the thread id, so let's clean up our default console pipeline

    (p/set-policy! :console '[(dissoc :thread-id) sink/console-log])

    (p/probe #{:debug} :value 10)
    2013-05-10T22:59.212 user:1 {:value 10}
	=> nil

What does our global configuration look like now?

    (clojure.pprint/pprint @p/policies)
	=> {:console [(dissoc :thread-id) sink/console-log]
        :default [#<core$identity clojure.core$identity@789df8c>]}

    (clojure.pprint/pprint @p/config)
    => {user {#{:debug} :console}}

So we can create probe points anywhere in our code, and do anything we want
with the combination of lexical and dynamic context.  Of course good functional
programs already have wonderful probe points available, they're called functions.

    (def testprobe [a b]
	  (+ a b))

    (p/probe-fn! 'testprobe)

	(testprobe 1 2)
    => 3

    (p/set-config! 'user :enter-fn :console)

	(testprobe 1 2)
	2013-05-10T23:28.627 user:1 {:args (1 2), :fn :enter, :fname testprobe}
	=> 3

	(p/remove-config! 'user :enter-fn)
    (p/set-config! 'user :fn :console)
	(defn testprobe [a b] (+ 1 a b))
	(testprobe 1 2)
    2013-05-10T23:30.522 user:1 {:args (1 2), :fn :enter, :fname testprobe}
    2013-05-10T23:30.522 user:1 {:return 4, :args (1 2), :fn :exit, :fname testprobe}
	=> 3 ;; Wrapping survives redefinition

	(p/remove-config! 'user :fn)
    (p/set-config! 'user :exit-fn
       '[(select-keys [:fname :return :args]) sink/console-raw])

    (testprobe 1 2)
    {:fname testprobe, :args (1 2), :return 4}
    => 4

This demonstrates generating test vectors from runtime for any
function simply by probing the function and appropriately filtering
the result.  What if we want to capture these vectors into memory for
replay later?

    (def my-tests (atom nil))
    (p/set-config! 'user :exit-fn
	   '[(select-keys [:fname :return :args]) (sink/memory my-tests)])
    
    (testprobe 1 4) 
    (testprobe 1 5)
    => 7

    @my-tests
    ({:args (1 5), :return 7, :fname testprobe} {:args (1 4), :return 6, :fname testprobe})

Of course the flood of vectors could get overwhelming in a real system, even if you just turn on this ability for a short while.  We can capture a random sub-sample of the traced data:

    (p/set-config! 'user :exit-fn
	   '[(policy/random-sample 0.01)
         (select-keys [:fname :return :args]) 
         (sink/memory my-tests)])

We can used a fixed length in-memory queue to keep the last N items

    (def my-tests (sink/make-memory))
    (p/set-config! 'user :exit-fn
	   '[(policy/random-sample 0.01)
         (select-keys [:fname :return :args]) 
         (sink/fixed-memory my-tests 5)])

    (dotimes [i 10]
      (testprobe i 10))

    (clojure.pprint/pprint (seq @my-tests))
    => ({:args (5 10), :return 16, :fname testprobe} 
        {:args (7 10), :return 17, :fname testprobe} 
        {:args (8 10), :return 18, :fname testprobe} 
        {:args (9 10), :return 19, :fname testprobe} 
        {:args (10 10), :return 20, :fname testprobe})

We can also select for specific functions in the namespace:
        
    (p/set-config! 'user :exit-fn
	   '[(policy/select-fn testprobe)
         (select-keys [:fname :return :args]) 
         (sink/memory my-tests)])
    
Or only collect data when there are failures:

    (p/remove-config! 'user :exit-fn)
    (p/set-config! 'user :except-fn
	   '[(select-keys [:ts :fname :return :args])
         sink/console-raw])

    (reset! my-tests (sink/make-memory))

    (defn testprobe [a b] (if (= a 2) (throw (Exception. "oops")) (+ a b 1)))
    (testprobe 1 3)
    => 5
	(testprobe 2 3)
    {:args (2 3), :fname testprobe}
    ; Evaluation aborted.

## Discussion

One of my internal debates in the first version of this was whether to use
classic ring middleware that uses function composition rather than linear
sequences of symbols or the use of named policies to do composition.  It's
obviously more flexible, but it's also harder to reason about.  

## Future Work

Documented here are some 

### Future Tasks (Minor)

* Add a policy to extract a clean stack trace from an exception
* Grab the stack state from a probe point (for profiling later)
* Add a simple dump of clojure data to a logfile, walk the data
  structure to ensure nothing unserializable causes errors?
* Add higher level targeted function tracing / collection facilities
  (e.g. trace 100 input/result vectors from function f or namespace ns)

### Future Tasks (Major)

* Probe log messages - Most systems will have legacy libraries that
     use one of the Java logging systems.  Create some namespaces that
     allow for injecting these log messages into clj-probe middleware.
* Adapt the function probes to also collect profile information

   	 