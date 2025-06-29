---
title: 02. service manager
author: ppdl
date: 2025-07-06 21:55:27
categories: [systemd]
mermaid: true
tags: [systemd,service-manager]     # TAG names should always be lowercase
---

## service-manager

systemd-service manager의 Job실행은 세가지의 구성요소, Unit, Job, Transaction, 그리고 하나의 Runqueue로 구성된다.
![service-manager](/assets/drawio/service-manager.svg)

### 1. Unit
   - service/target/mount/socket/... 등등

### 2. Job
   - **JobType**: 해당 Job이 Unit에 대해 수행해야하는 일
        - start/stop/restart/reload/...
   - **JobMode**: Job이 Runqueue에 들어갈 때, 이미 queue에 존재하는 Job을 다루는 방법, transaction단위로 설정됨 ([man systemctl](https://www.freedesktop.org/software/systemd/man/latest/systemctl.html#--job-mode=))
      ```
      enum JobMode {
        JOB_FAIL,                /* Fail if a conflicting job is already queued */
        JOB_REPLACE,             /* Replace an existing conflicting job */
        JOB_REPLACE_IRREVERSIBLY,/* Like JOB_REPLACE + produce irreversible jobs */
        JOB_ISOLATE,             /* Start a unit, and stop all others */
        JOB_FLUSH,               /* Flush out all other queued jobs when queueing this one */
        JOB_IGNORE_DEPENDENCIES, /* Ignore both requirement and ordering dependencies */
        JOB_IGNORE_REQUIREMENTS, /* Ignore requirement dependencies */
        JOB_TRIGGERING,          /* Adds TRIGGERED_BY dependencies to the same transaction */
        JOB_RESTART_DEPENDENCIES,/* A "start" job for the specified unit becomes "restart" for depending units */
        _JOB_MODE_MAX,
        _JOB_MODE_INVALID = -EINVAL,
      };
      ```
      {:file='src/core/job.h'}
   - Journal log(loglevel=debug)에서 보통 아래와 같은 내용을 관찰 할 수 있고, 각각 Unit/JobType/JobMode 를 나타낸다.

     ```Trying to enqueue job XXX.service/start/replace``` 

   - 예시
     ```
     [Unit]
     Description=foo service

     [Service]
     Type=oneshot
     ExecStartPre=/usr/bin/sleep 10
     ExecStart=/bin/true
     ```
     {:file='foo.service'}

     1. ``systemctl start foo.service``<br/>
        ``systemctl stop foo.service``

        ```
        foo.service: Trying to enqueue job foo.service/start/replace
        foo.service: Installed new job foo.service/start as 3218
        foo.service: Enqueued job foo.service/start as 3218
        foo.service: About to execute /usr/bin/sleep 10
        foo.service: Forked /usr/bin/sleep as 11963
        ...
        foo.service: Trying to enqueue job foo.service/stop/replace
        foo.service: Job 3218 foo.service/start finished, result=canceled
        foo.service: Installed new job foo.service/stop as 3287
        foo.service: Enqueued job foo.service/stop as 3287
        ```
        {:file='journalctl'}
     2. ``systemctl start foo.service``<br/>
        ``systemctl stop foo.service --job-mode=fail``

        ```
        foo.service: Trying to enqueue job foo.service/start/replace
        foo.service: Installed new job foo.service/start as 3288
        foo.service: Enqueued job foo.service/start as 3288
        foo.service: About to execute /usr/bin/sleep 10
        foo.service: Forked /usr/bin/sleep as 12191
        ...
        foo.service: Trying to enqueue job foo.service/stop/fail
        Requested transaction contradicts existing jobs: Transaction for foo.service/stop is destructive
        (foo.service has 'start' job queued, but 'stop' is included in transaction).
        ```
        {:file='journalctl'}
     3. ``reboot``
        ```
        [Unit]
        Description=bar
        Conflicts=reboot.target
        DefaultDependencies=no
     
        [Service]
        ExecStart=/bin/true
        ```
        {:file='reboot.target.wants/bar.service -> bar.service'}
        ```
        reboot.target: Trying to enqueue job reboot.target/start/replace-irreversibly
        Added job reboot.target/start to transaction
        ...
        Pulling in bar.service/start from reboot.target/start
        Added job bar.service/start to transaction.
        Added job reboot.target/stop to transaction.
        ...
        reboot.target: Fixing conflicting jobs reboot.target/stop,reboot.target/start by deleting job reboot.target/stop
        bar.service: Deleting job bar.service/start as dependency of job reboot.target/stop
        ...
        Found redundant job bar.service/stop, dropping from transaction
        reboot.target: Installed new job reboot.target/start as 1513
        reboot.target: Enqueued job reboot.target/start as 1513
        ```
        {:file='journalctl'}


### 3. Transaction
   - Job의 **Requirement Dependency**에 따라 연계된 여러 Job들의 chain
      - Requirement Dependency: Wants=, Requires=, ...
   - **Anchor Job** : Transaction을 만들기 시작하는 최초의 Job
      - systemd 시작 시 default.target
      - systemctl start/stop/... <Unit> 에 명시되는 Job
      - dbus activation / socket activation으로 systemd가 activate시키는 Unit
      - 등등...

### 4. Run queue
   - Priority queue
      - ```
        int unit_compare_priority(Unit *a, Unit *b) {
            int ret;

            ret = CMP(a->type, b->type);
            if (ret != 0)
                    return -ret;

            ret = CMP(unit_get_cpu_weight(a), unit_get_cpu_weight(b));
            if (ret != 0)
                    return -ret;

            ret = CMP(unit_get_nice(a), unit_get_nice(b));
            if (ret != 0)
                    return ret;

            return strcmp(a->id, b->id);
        }
        ```
        {:file='src/core/unit.c'}
           1. Unit Type (내림차순, UNIT_SCOPE -> UNIT_SERVICE)
              ```
              typedef enum UnitType {
                        UNIT_SERVICE,
                        UNIT_MOUNT,
                        UNIT_SWAP,
                        UNIT_SOCKET,
                        UNIT_TARGET,
                        UNIT_DEVICE,
                        UNIT_AUTOMOUNT,
                        UNIT_TIMER,
                        UNIT_PATH,
                        UNIT_SLICE,
                        UNIT_SCOPE,
                        _UNIT_TYPE_MAX,
                        _UNIT_TYPE_INVALID = -EINVAL,
                        _UNIT_TYPE_ERRNO_MAX = -ERRNO_MAX, /* Ensure the whole errno range fits into this enum */
                } UnitType;
              ```
              {:file='src/basic/unit-def.h'}
           2. cpu weight (내림차순)
           3. nice (오름차순)
           4. 알파벳 (오름차순)

   - Transaction의 Job들이 queueing되어 **Ordering Dependency**에 의해 실행
      - Ordering Dependency: Before=, After=
