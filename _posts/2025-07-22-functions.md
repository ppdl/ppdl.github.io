---
title: 03. functions
author: ppdl
date: 2025-07-22 22:04:35
categories: [systemd]
mermaid: true
tags: [systemd]     # TAG names should always be lowercase
---

## do_queue_default_job()
```c
static int do_queue_default_job(
                Manager *m,
                const char **ret_error_message) {

        _cleanup_(sd_bus_error_free) sd_bus_error error = SD_BUS_ERROR_NULL;
        const char *unit;
        Job *job;
        Unit *target;
        int r;

        if (arg_default_unit)
                unit = arg_default_unit; // systemd --unit=XXX
        else if (in_initrd())
                unit = SPECIAL_INITRD_TARGET; // "initrd.target"
        else
                unit = SPECIAL_DEFAULT_TARGET; // "default.target"

        log_debug("Activating default unit: %s", unit);

        // "default.target"으로 실제 Unit 구조체를 가져옴, &target
        r = manager_load_startable_unit_or_warn(m, unit, NULL, &target);
        [...]

        // default.target/start/isolate 로 job을 추가함
        r = manager_add_job(m, JOB_START, target, JOB_ISOLATE, NULL, &error, &job);
        [...]

        log_info("Queued %s job for default target %s.",
                 job_type_to_string(job->type),
                 unit_status_string(job->unit, NULL));

        m->default_unit_job_id = job->id;

        return 0;
}

```
{:file='src/core/main.c'}


## manager_add_job()
```c
int manager_add_job(
                Manager *m,
                JobType type,
                Unit *unit,
                JobMode mode,
                Set *affected_jobs,
                sd_bus_error *error,
                Job **ret) {

        _cleanup_(transaction_abort_and_freep) Transaction *tr = NULL;
        int r;

        [...]

        log_unit_debug(unit, "Trying to enqueue job %s/%s/%s", unit->id, job_type_to_string(type), job_mode_to_string(mode));
       
        [...]

        // 새로운 transaction 객체 생성
        tr = transaction_new(mode == JOB_REPLACE_IRREVERSIBLY);
        if (!tr)
                return -ENOMEM;

        // transaction에 unit/JobType/JobMode 및 그requirement dependency로 연결된 unit들을 추가
        r = transaction_add_job_and_dependencies(
                        tr,
                        type,
                        unit,
                        /* by= */ NULL,
                        TRANSACTION_MATTERS |
                        (IN_SET(mode, JOB_IGNORE_DEPENDENCIES, JOB_IGNORE_REQUIREMENTS) ? TRANSACTION_IGNORE_REQUIREMENTS : 0) |
                        (mode == JOB_IGNORE_DEPENDENCIES ? TRANSACTION_IGNORE_ORDER : 0) |
                        (mode == JOB_RESTART_DEPENDENCIES ? TRANSACTION_PROPAGATE_START_AS_RESTART : 0),
                        error);
        if (r < 0)
                return r;

        // JobMode가 isolate일 경우 transaction과 requirement dependency가 없는 unit들을 stop하는 job을 추가
        if (mode == JOB_ISOLATE) {
                r = transaction_add_isolate_jobs(tr, m);
                if (r < 0)
                        return r;
        }

        // JobMode가 triggering경우(JobType이 Stop일 경우만 가능) unit과 triggring관계의 unit들을 stop하는 job을 추가
        // 예시)
        // foo.socket과 foo.service가 둘다 active상태이고, triggering관계가 있을 때(e.g., socket activation),
        // systemctl stop foo.service 하면 foo.service만 stop되고 foo.socket은 running->listening상태로 바뀌지만
        // systemctl stop foo.service --job-mode=triggering으로 하면 둘 다 stop됨
        if (mode == JOB_TRIGGERING) {
                r = transaction_add_triggering_jobs(tr, unit);
                if (r < 0)
                        return r;
        }

        // 만들어진 transaction을 manager의 runqueue에 추가
        r = transaction_activate(tr, m, mode, affected_jobs, error);
        if (r < 0)
                return r;

        log_unit_debug(unit,
                       "Enqueued job %s/%s as %u", unit->id,
                       job_type_to_string(type), (unsigned) tr->anchor_job->id);

        if (ret)
                *ret = tr->anchor_job;

        tr = transaction_free(tr);
        return 0;
}
```
{:file='src/core/manager.c'}

## transaction_add_job_and_dependencies()
```c
int transaction_add_job_and_dependencies(
                Transaction *tr,
                JobType type,
                Unit *unit,
                Job *by, // 누구에 의해 해당 job이 추가되었는지 명시, 재귀적으로 호출될 때 세팅됨
                TransactionAddFlags flags,
                sd_bus_error *e) {

        bool is_new;
        Job *ret;
        int r;

        [...]

        if (by)
                log_trace("Pulling in %s/%s from %s/%s", unit->id, job_type_to_string(type), by->unit->id, job_type_to_string(by->type));

        [...]

        // transaction에 해당 job을 추가
        /* First add the job. */
        ret = transaction_add_one_job(tr, type, unit, &is_new);
        if (!ret)
                return -ENOMEM;

        [...]

        /* Then, add a link to the job. */
        if (by) {
                // by job과 현재 job과의 dependency 추가 -> 해당 구조체 용도 추가
                if (!job_dependency_new(by, ret, FLAGS_SET(flags, TRANSACTION_MATTERS), FLAGS_SET(flags, TRANSACTION_CONFLICTS)))
                        return -ENOMEM;
        } else {
                // by == NULL 이면, 해당 job은 anchor job
                /* If the job has no parent job, it is the anchor job. */
                assert(!tr->anchor_job);
                tr->anchor_job = ret;
        }

        // 이러한 경우는 해당 job의 requirement dependency를 추가하지 않고 종료
        if (!is_new || FLAGS_SET(flags, TRANSACTION_IGNORE_REQUIREMENTS) || type == JOB_NOP)
                return 0;

        _cleanup_set_free_ Set *following = NULL;
        Unit *dep;

        // 두가지 경우 Device unit과 Swap unit에만 해당하는 동작
        // following set이란
        //   - Device unit일 경우 동일한 sysfs node에 대해 서로 다른 Device unit을 전부 transaction에 추가
        //   - Swap unit일 경우 동일한 devnode에 대해 서로 다른 Swap unit을 전부 transaction에 추가
        /* If we are following some other unit, make sure we add all dependencies of everybody following. */
        if (unit_following_set(ret->unit, &following) > 0)
                SET_FOREACH(dep, following) {
                        r = transaction_add_job_and_dependencies(tr, type, dep, ret, flags & TRANSACTION_IGNORE_ORDER, e);
                        if (r < 0) {
                                log_unit_full_errno(dep, r == -ERFKILL ? LOG_INFO : LOG_WARNING, r,
                                                    "Cannot add dependency job, ignoring: %s",
                                                    bus_error_message(e, r));
                                sd_bus_error_free(e);
                        }
                }

        /* Finally, recursively add in all dependencies. */
        if (IN_SET(type, JOB_START, JOB_RESTART)) {
                UNIT_FOREACH_DEPENDENCY(dep, ret->unit, UNIT_ATOM_PULL_IN_START) {
                        r = transaction_add_job_and_dependencies(tr, JOB_START, dep, ret, TRANSACTION_MATTERS | (flags & TRANSACTION_IGNORE_ORDER), e);
                        if (r < 0) {
                                if (r != -EBADR) /* job type not applicable */
                                        goto fail;

                                sd_bus_error_free(e);
                        }
                }

                UNIT_FOREACH_DEPENDENCY(dep, ret->unit, UNIT_ATOM_PULL_IN_START_IGNORED) {
                        r = transaction_add_job_and_dependencies(tr, JOB_START, dep, ret, flags & TRANSACTION_IGNORE_ORDER, e);
                        if (r < 0) {
                                /* unit masked, job type not applicable and unit not found are not considered
                                 * as errors. */
                                log_unit_full_errno(dep,
                                                    IN_SET(r, -ERFKILL, -EBADR, -ENOENT) ? LOG_DEBUG : LOG_WARNING,
                                                    r, "Cannot add dependency job, ignoring: %s",
                                                    bus_error_message(e, r));
                                sd_bus_error_free(e);
                        }
                }

                UNIT_FOREACH_DEPENDENCY(dep, ret->unit, UNIT_ATOM_PULL_IN_VERIFY) {
                        r = transaction_add_job_and_dependencies(tr, JOB_VERIFY_ACTIVE, dep, ret, TRANSACTION_MATTERS | (flags & TRANSACTION_IGNORE_ORDER), e);
                        if (r < 0) {
                                if (r != -EBADR) /* job type not applicable */
                                        goto fail;

                                sd_bus_error_free(e);
                        }
                }

                UNIT_FOREACH_DEPENDENCY(dep, ret->unit, UNIT_ATOM_PULL_IN_STOP) {
                        r = transaction_add_job_and_dependencies(tr, JOB_STOP, dep, ret, TRANSACTION_MATTERS | TRANSACTION_CONFLICTS | (flags & TRANSACTION_IGNORE_ORDER), e);
                        if (r < 0) {
                                if (r != -EBADR) /* job type not applicable */
                                        goto fail;

                                sd_bus_error_free(e);
                        }
                }

                UNIT_FOREACH_DEPENDENCY(dep, ret->unit, UNIT_ATOM_PULL_IN_STOP_IGNORED) {
                        r = transaction_add_job_and_dependencies(tr, JOB_STOP, dep, ret, flags & TRANSACTION_IGNORE_ORDER, e);
                        if (r < 0) {
                                log_unit_warning(dep,
                                                 "Cannot add dependency job, ignoring: %s",
                                                 bus_error_message(e, r));
                                sd_bus_error_free(e);
                        }
                }
        }

        if (IN_SET(type, JOB_RESTART, JOB_STOP) || (type == JOB_START && FLAGS_SET(flags, TRANSACTION_PROPAGATE_START_AS_RESTART))) {
                bool is_stop = type == JOB_STOP;

                UNIT_FOREACH_DEPENDENCY(dep, ret->unit, is_stop ? UNIT_ATOM_PROPAGATE_STOP : UNIT_ATOM_PROPAGATE_RESTART) {
                        /* We propagate RESTART only as TRY_RESTART, in order not to start dependencies that
                         * are not around. */
                        JobType nt;

                        nt = job_type_collapse(is_stop ? JOB_STOP : JOB_TRY_RESTART, dep);
                        if (nt == JOB_NOP)
                                continue;

                        r = transaction_add_job_and_dependencies(tr, nt, dep, ret, TRANSACTION_MATTERS | (flags & TRANSACTION_IGNORE_ORDER), e);
                        if (r < 0) {
                                if (r != -EBADR) /* job type not applicable */
                                        return r;

                                sd_bus_error_free(e);
                        }
                }

                /* Process UNIT_ATOM_PROPAGATE_STOP_GRACEFUL (PropagatesStopTo=) units. We need to wait until
                 * all other dependencies are processed, i.e. we're the anchor job or already in the recursion
                 * that handles it. */
                if (!by || FLAGS_SET(flags, TRANSACTION_PROCESS_PROPAGATE_STOP_GRACEFUL))
                        UNIT_FOREACH_DEPENDENCY(dep, ret->unit, UNIT_ATOM_PROPAGATE_STOP_GRACEFUL) {
                                JobType nt;
                                Job *j;

                                j = hashmap_get(tr->jobs, dep);
                                nt = job_type_propagate_stop_graceful(j);

                                if (nt == JOB_NOP)
                                        continue;

                                r = transaction_add_job_and_dependencies(tr, nt, dep, ret, TRANSACTION_MATTERS | (flags & TRANSACTION_IGNORE_ORDER) | TRANSACTION_PROCESS_PROPAGATE_STOP_GRACEFUL, e);
                                if (r < 0) {
                                        if (r != -EBADR) /* job type not applicable */
                                                return r;

                                        sd_bus_error_free(e);
                                }
                        }
        }

        if (type == JOB_RELOAD)
                transaction_add_propagate_reload_jobs(tr, ret->unit, ret, flags & TRANSACTION_IGNORE_ORDER);

        /* JOB_VERIFY_ACTIVE requires no dependency handling */

        return 0;

fail:
        [...]
}
```
{:file='src/core/transaction.c'}

 - atom_map
   ```c
   static const UnitDependencyAtom atom_map[_UNIT_DEPENDENCY_MAX] = {
        /* A table that maps high-level dependency types to low-level dependency "atoms". The latter actually
         * describe specific facets of dependency behaviour. The former combine them into one user-facing
         * concept. Atoms are a bit mask, though a bunch of dependency types have only a single bit set.
         *
         * Typically when the user configures a dependency they go via dependency type, but when we act on
         * them we go by atom.
         *
         * NB: when you add a new dependency type here, make sure to also add one to the (best-effort)
         * reverse table in unit_dependency_from_unique_atom() further down. */

        [UNIT_REQUIRES]               = UNIT_ATOM_PULL_IN_START |
                                        UNIT_ATOM_RETROACTIVE_START_REPLACE |
                                        UNIT_ATOM_ADD_STOP_WHEN_UNNEEDED_QUEUE |
                                        UNIT_ATOM_ADD_DEFAULT_TARGET_DEPENDENCY_QUEUE,

        [UNIT_REQUISITE]              = UNIT_ATOM_PULL_IN_VERIFY |
                                        UNIT_ATOM_ADD_STOP_WHEN_UNNEEDED_QUEUE |
                                        UNIT_ATOM_ADD_DEFAULT_TARGET_DEPENDENCY_QUEUE,

        [UNIT_WANTS]                  = UNIT_ATOM_PULL_IN_START_IGNORED |
                                        UNIT_ATOM_RETROACTIVE_START_FAIL |
                                        UNIT_ATOM_ADD_STOP_WHEN_UNNEEDED_QUEUE |
                                        UNIT_ATOM_ADD_DEFAULT_TARGET_DEPENDENCY_QUEUE,

        [UNIT_BINDS_TO]               = UNIT_ATOM_PULL_IN_START |
                                        UNIT_ATOM_RETROACTIVE_START_REPLACE |
                                        UNIT_ATOM_CANNOT_BE_ACTIVE_WITHOUT |
                                        UNIT_ATOM_ADD_STOP_WHEN_UNNEEDED_QUEUE |
                                        UNIT_ATOM_ADD_DEFAULT_TARGET_DEPENDENCY_QUEUE,

        [UNIT_PART_OF]                = UNIT_ATOM_ADD_DEFAULT_TARGET_DEPENDENCY_QUEUE,

        [UNIT_UPHOLDS]                = UNIT_ATOM_PULL_IN_START_IGNORED |
                                        UNIT_ATOM_RETROACTIVE_START_REPLACE |
                                        UNIT_ATOM_ADD_START_WHEN_UPHELD_QUEUE |
                                        UNIT_ATOM_ADD_STOP_WHEN_UNNEEDED_QUEUE |
                                        UNIT_ATOM_ADD_DEFAULT_TARGET_DEPENDENCY_QUEUE,

        [UNIT_REQUIRED_BY]            = UNIT_ATOM_PROPAGATE_STOP |
                                        UNIT_ATOM_PROPAGATE_RESTART |
                                        UNIT_ATOM_PROPAGATE_START_FAILURE |
                                        UNIT_ATOM_PINS_STOP_WHEN_UNNEEDED |
                                        UNIT_ATOM_DEFAULT_TARGET_DEPENDENCIES,

        [UNIT_REQUISITE_OF]           = UNIT_ATOM_PROPAGATE_STOP |
                                        UNIT_ATOM_PROPAGATE_RESTART |
                                        UNIT_ATOM_PROPAGATE_START_FAILURE |
                                        UNIT_ATOM_PROPAGATE_INACTIVE_START_AS_FAILURE |
                                        UNIT_ATOM_PINS_STOP_WHEN_UNNEEDED |
                                        UNIT_ATOM_DEFAULT_TARGET_DEPENDENCIES,

        [UNIT_WANTED_BY]              = UNIT_ATOM_DEFAULT_TARGET_DEPENDENCIES |
                                        UNIT_ATOM_PINS_STOP_WHEN_UNNEEDED,

        [UNIT_BOUND_BY]               = UNIT_ATOM_RETROACTIVE_STOP_ON_STOP |
                                        UNIT_ATOM_PROPAGATE_STOP |
                                        UNIT_ATOM_PROPAGATE_RESTART |
                                        UNIT_ATOM_PROPAGATE_START_FAILURE |
                                        UNIT_ATOM_PINS_STOP_WHEN_UNNEEDED |
                                        UNIT_ATOM_ADD_CANNOT_BE_ACTIVE_WITHOUT_QUEUE |
                                        UNIT_ATOM_DEFAULT_TARGET_DEPENDENCIES,

        [UNIT_UPHELD_BY]              = UNIT_ATOM_START_STEADILY |
                                        UNIT_ATOM_DEFAULT_TARGET_DEPENDENCIES |
                                        UNIT_ATOM_PINS_STOP_WHEN_UNNEEDED,

        [UNIT_CONSISTS_OF]            = UNIT_ATOM_PROPAGATE_STOP |
                                        UNIT_ATOM_PROPAGATE_RESTART,

        [UNIT_CONFLICTS]              = UNIT_ATOM_PULL_IN_STOP |
                                        UNIT_ATOM_RETROACTIVE_STOP_ON_START,

        [UNIT_CONFLICTED_BY]          = UNIT_ATOM_PULL_IN_STOP_IGNORED |
                                        UNIT_ATOM_RETROACTIVE_STOP_ON_START |
                                        UNIT_ATOM_PROPAGATE_STOP_FAILURE,

        [UNIT_PROPAGATES_STOP_TO]     = UNIT_ATOM_RETROACTIVE_STOP_ON_STOP |
                                        UNIT_ATOM_PROPAGATE_STOP_GRACEFUL,

        /* These are simple dependency types: they consist of a single atom only */
        [UNIT_ON_FAILURE]             = UNIT_ATOM_ON_FAILURE,
        [UNIT_ON_SUCCESS]             = UNIT_ATOM_ON_SUCCESS,
        [UNIT_ON_FAILURE_OF]          = UNIT_ATOM_ON_FAILURE_OF,
        [UNIT_ON_SUCCESS_OF]          = UNIT_ATOM_ON_SUCCESS_OF,
        [UNIT_BEFORE]                 = UNIT_ATOM_BEFORE,
        [UNIT_AFTER]                  = UNIT_ATOM_AFTER,
        [UNIT_TRIGGERS]               = UNIT_ATOM_TRIGGERS,
        [UNIT_TRIGGERED_BY]           = UNIT_ATOM_TRIGGERED_BY,
        [UNIT_PROPAGATES_RELOAD_TO]   = UNIT_ATOM_PROPAGATES_RELOAD_TO,
        [UNIT_JOINS_NAMESPACE_OF]     = UNIT_ATOM_JOINS_NAMESPACE_OF,
        [UNIT_REFERENCES]             = UNIT_ATOM_REFERENCES,
        [UNIT_REFERENCED_BY]          = UNIT_ATOM_REFERENCED_BY,
        [UNIT_IN_SLICE]               = UNIT_ATOM_IN_SLICE,
        [UNIT_SLICE_OF]               = UNIT_ATOM_SLICE_OF,

        /* These are dependency types without effect on our state engine. We maintain them only to make
         * things discoverable/debuggable as they are the inverse dependencies to some of the above. As they
         * have no effect of their own, they all map to no atoms at all, i.e. the value 0. */
        [UNIT_RELOAD_PROPAGATED_FROM] = 0,
        [UNIT_STOP_PROPAGATED_FROM]   = 0,
   };

   ```
   {:file='src/core/unit-dependency-atom.c'}
