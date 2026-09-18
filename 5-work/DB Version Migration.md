1. Validate the new version locally using docker and running the full test suite
	- you may get a docker error when upgrading the instance if you have an existing docker volume on the old version. wipe and recreate the volume
2. Create new parameter group
	```bash
	export STACK = <ENV_TO_UPDATE>
	```
	```bash
	AWS_PROFILE=work aws rds create-db-parameter-group \
        --db-parameter-group-name macro-db-parameter-group-$STACK-pg18 \
        --db-parameter-group-family postgres18 \
        --description "$STACK params for postgres 18"
	```
3. Apply the new parameters
	```bash
AWS_PROFILE=work aws rds modify-db-parameter-group \
        --db-parameter-group-name macro-db-parameter-group-$STACK-pg18 \
        --parameters \
          "ParameterName=auto_explain.log_analyze,ParameterValue=1,ApplyMethod=pending-reboot" \
          "ParameterName=auto_explain.log_buffers,ParameterValue=1,ApplyMethod=pending-reboot" \
          "ParameterName=auto_explain.log_format,ParameterValue=json,ApplyMethod=pending-reboot" \
          "ParameterName=auto_explain.log_min_duration,ParameterValue=1000,ApplyMethod=pending-reboot" \
          "ParameterName=auto_explain.log_nested_statements,ParameterValue=1,ApplyMethod=pending-reboot" \
          "ParameterName=auto_explain.log_timing,ParameterValue=0,ApplyMethod=pending-reboot" \
          "ParameterName=auto_explain.log_triggers,ParameterValue=1,ApplyMethod=pending-reboot" \
          "ParameterName=auto_explain.log_verbose,ParameterValue=1,ApplyMethod=pending-reboot" \
          "ParameterName=auto_explain.sample_rate,ParameterValue=1,ApplyMethod=pending-reboot" \
          "ParameterName=checkpoint_timeout,ParameterValue=900,ApplyMethod=pending-reboot" \
          "ParameterName=effective_io_concurrency,ParameterValue=256,ApplyMethod=pending-reboot" \
          "ParameterName=idle_in_transaction_session_timeout,ParameterValue=300000,ApplyMethod=pending-reboot" \
          "ParameterName=log_temp_files,ParameterValue=10240,ApplyMethod=pending-reboot" \
          "ParameterName=max_wal_size,ParameterValue=16384,ApplyMethod=pending-reboot" \
          "ParameterName=min_wal_size,ParameterValue=4096,ApplyMethod=pending-reboot" \
          "ParameterName=random_page_cost,ParameterValue=1.1,ApplyMethod=pending-reboot" \
          "ParameterName=shared_preload_libraries,ParameterValue=\"pg_stat_statements,auto_explain\",ApplyMethod=pending-reboot" \
          "ParameterName=vacuum_cost_page_miss,ParameterValue=10,ApplyMethod=pending-reboot" \
          "ParameterName=work_mem,ParameterValue=16384,ApplyMethod=pending-reboot"
	```
4. Trigger the upgrade
	```bash
	AWS_PROFILE=work aws rds modify-db-instance \
        --db-instance-identifier macro-db-$STACK \
        --engine-version 18.3 \
        --db-parameter-group-name macro-db-parameter-group-$STACK-pg18 \
        --allow-major-version-upgrade \
        --apply-immediately
	```
5. Validate the parameter group is updated
	```bash
	AWS_PROFILE=work aws rds describe-db-instances --db-instance-identifier macro-db-$STACK \
        --query "DBInstances[0].{status:DBInstanceStatus,version:EngineVersion,pending:PendingModifiedValues,paramGroups:DBParameterGroups}"
	```
6. Update pulumi manually to sync with new values
- TODO: SYNC PULUMI