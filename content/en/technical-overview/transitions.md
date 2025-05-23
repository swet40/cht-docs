---
title: "Sentinel Transitions"
linkTitle: "Sentinel Transitions"
weight: 10
description: >
  Overview of transitions and database documents change
relatedContent: >  
  building/reference/app-settings/transitions
aliases:
   - /core/overview/transitions/
---

A transition is a Javascript code that runs when a document is changed.  A
transition can edit the changed doc or do anything server-side code can do for
that matter.

Transitions are run in series, not in parallel:

* For a given change, you can expect one transition to be finished before the
  next runs.

* You can expect one change to be fully processed by all transitions before
  the next starts being processed.

Transitions obey the following rules:

* has a `filter(doc)` function that accepts the changed document as an argument and
  returns `true` if it needs to run/be applied.

* a `onMatch(change, db, auditDb, callback)` function that will run on changes
  that pass the filter.
  
* can have an `init()` function to do any required setup and throw Errors on invalid
  configuration.

* has an `onChange(change, db, audit, callback)` function that makes changes to
  the `change.doc` reference (copying is discouraged). `db` and `audit` are
  handled to let you query those DBs. You can learn more about `callback` in subsequent sections.

* It is not necessary for an individual transition to save the changes made to `change.doc` to the DB, the doc will be saved once all the transitions have been edited.
If an individual transition saves the document provided at `change.doc`, it takes responsibility for re-attaching the newly saved document (with new seq etc) at `change.doc`

* guarantees the consistency of a document.

* runs serially and in any order.  A transition is free to make async calls but
  the next transition will only run after the previous transition's callback
  is called. If your transition is dependent on another transition then it will
  run on the next change.  Code your transition to support two changes rather
  than require a specific order.  You can optimize your ordering but don't
  require it.  This keeps configuration simpler.

* is repeatable, it can run multiple times on the same document without
  negative effect.  You can use the `transitions` property on a document to
  determine if a transition has run.


Callback arguments:

* callback(err, needsSaving)

   `needsSaving` is true if the `change.doc` needs to be saved to the DB by the transition runner. For instance, if the transition has edited the `change.doc` in memory.
   `err` if truthy, the error will be added to the `changes.doc` in memory. (Note that if `needsSaving` is falsy, the doc will not be saved, so that error will not be persisted).

Regardless of whether the doc is saved or not, the transitions will all be run (unless one crashes).

When your transition encounters an error, there are different ways to deal with it. You can :
- finish your transition with `callback(someError, true)`. This will save the error to `change.doc`.
- finish your transition with `callback(someError, false)`. The error will be logged to the sentinel log. This will not save the error on the `change.doc`, so there will be no record that this transition ran. That particular `change` will not go through transitions again, but if the same doc has another change in the future since there is no record of the erroring transition having run, it will be rerun.
- crash sentinel. When sentinel restarts, since that `change` did not record successful processing, it will be reprocessed. Transitions that did not save anything to the `change.doc` will be rerun.
