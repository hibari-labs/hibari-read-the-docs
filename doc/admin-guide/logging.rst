Hibari Application Logging
==========================

NOTE: This chapter is outdated and will be rewritten by Hibari v0.6
release. Hibari now uses link:https://github.com/basho/lager#readme[Basho Lager]
for logging and the default location of the log files is:
`<HIBARI_HOME>/logs/`

The Hibari application log records application-related alerts,
warnings, and informational messages, as well as trace messages for
debugging. By default the application log is written to this file:

  <HIBARI_HOME>/var/log/gdss-app.log

=== Format of the Hibari Application Log

Each log entry in the Hibari application log is composed of these
fields in this order, with vertical bar delimitation:

  <PID>|<<ERLANGPID>>|<DATETIME>|<MODULE>|<LEVEL>|<MESSAGECODE>|<MESSAGE>

This Hibari application log entry format is not configurable.  Each of
these application log entry fields is described in the table that
follows.  The ``Position'' column indicates the position of the field
within a log entry.

[options="header",cols="^,^m,<"]
|=========
| Position | Field | Description
| 1 | <PID> | System-assigned process identifier (PID) of the process that generated the log message.
| 2 | <ERLANGPID> | Erlang process identifier.
| 3 | <DATETIME> | Timestamp in format `%Y%m%d%H%M%S`, where `%Y` = four digit year; `%m` = two digit month; `%d` = two digit date; `%H` = two digit hour; `%M` = two digit minute; and `%S` = two digit seconds. For example, `20081103230123`.
| 4 | <MODULE> | The internal component with which the message is associated. This field is set to a minimum length of 13 characters. If the module name is shorter than 13 characters, spaces will be appended to the module name so that the field reaches the 13 character minimum.
| 5 | <LEVEL> | The severity level of the message. The level will be one of the following: `ALERT`, a condition requiring immediate correction; `WARNG`, a warning message, indicating a potential problem; `INFO`, an informational message indicating normal activity, and requiring no action; `DEBUG`, a highly granular, process-descriptive message potentially of use when debugging the application.
| 6 | <MESSAGECODE> | Integer code assigned to all messages of severity level `INFO` or higher.  NOTE: This code is not yet defined in the Hibari open source release.
| 7 | <MESSAGE> | The message itself, describing the event that has occurred.
|=========

=== Application Log Example

Items written to the Hibari application log come from multiple sources:

* The Hibari OTP application
* Other OTP applications bundled with Hibari
* Other OTP applications within the Erlang runtime system,
  e.g. `kernel` and `sasl`.

The `<MESSAGE>` field is free-form text.  Application code can freely
add newline characters and various white-space padding wherever it
wishes.  However, the file format dictates that a newline character
(ASCII 10) appear only at the end of the entire app log message.

The Hibari error logger must therefore reformat the text of the
`<MESSAGE>` field to remove newlines and to remove whitespace
padding.  The result is not nearly as readable as the formatting
presented to the Erlang shell.  For example, within the shell, a
message can look like this:

  =PROGRESS REPORT==== 12-Apr-2010::17:49:22 ===
            supervisor: {local,sasl_safe_sup}
               started: [{pid,<0.43.0>},
                         {name,alarm_handler},
                         {mfa,{alarm_handler,start_link,[]}},
                         {restart_type,permanent},
                         {shutdown,2000},
                         {child_type,worker}]

Within the Hibari application log, however, the same message is
reformatted as line #2 below.  The reformatted version is much more
difficult for a human to read than the version above, but the purpose
of the app log file is to be machine-parsable, not human-parsable.

  8955|<0.54.0>|20100412174922|gmt_app      |INFO|2190301|start: normal []
  8955|<0.55.0>|20100412174922|SASL         |INFO|2199999|progress: [{supervisor,{local,gmt_sup}},{started,[{pid,<0.56.0>},{name,gmt_config_svr},{mfa,{gmt_config_svr,start_link,["../priv/central.conf"]}},{restart_type,permanent},{shutdown,2000},{child_type,worker}]}]
  8955|<0.55.0>|20100412174922|SASL         |INFO|2199999|progress: [{supervisor,{local,gmt_sup}},{started,[{pid,<0.57.0>},{name,gmt_tlog_svr},{mfa,{gmt_tlog_svr,start_link,[]}},{restart_type,permanent},{shutdown,2000},{child_type,worker}]}]
  8955|<0.36.0>|20100412174922|SASL         |INFO|2199999|progress: [{supervisor,{local,kernel_safe_sup}},{started,[{pid,<0.59.0>},{name,timer_server},{mfa,{timer,start_link,[]}},{restart_type,permanent},{shutdown,1000},{child_type,worker}]}]
  [...skipping ahead...]
  8955|<0.7.0>|20100412174923|SASL         |INFO|2199999|progress: [{application,gdss},{started_at,gdss_dev2@bb3}]
  8955|<0.98.0>|20100412174923|DEFAULT      |INFO|2199999|brick_sb: Admin Server not registered yet, retrying
  8955|<0.65.0>|20100412174923|SASL         |INFO|2199999|progress: [{supervisor,{local,brick_admin_sup}},{started,[{pid,<0.98.0>},{name,brick_sb},{mfa,{brick_sb,start_link,[]}},{restart_type,permanent},{shutdown,2000},{child_type,worker}]}]
  8955|<0.105.0>|20100412174924|DEFAULT      |INFO|2199999|top of init: bootstrap_copy1, [{implementation_module,brick_ets},{default_data_dir,"."}]
  8955|<0.105.0>|20100412174924|DEFAULT      |INFO|2199999|do_init_second_half: bootstrap_copy1
  8955|<0.79.0>|20100412174924|SASL         |INFO|2199999|progress: [{supervisor,{local,brick_brick_sup}},{started,[{pid,<0.105.0>},{name,bootstrap_copy1},{mfa,{brick_server,start_link,[bootstrap_copy1,[{default_data_dir,"."}]]}},{restart_type,temporary},{shutdown,2000},{child_type,worker}]}]
  8955|<0.105.0>|20100412174924|DEFAULT      |INFO|2199999|do_init_second_half: bootstrap_copy1 finished

== Examining Latency in Production (Internal Event Tracing)

The Hibari source code has been annotated with over 400 tracepoints,
and they give the developer and system administrator for tracing
events through Hibari's code. Those tracepoints are designed to be
extremely lightweight and can be enabled in production environment
without sacrificing performance.

Trace data can be collected via DTrace/SystemTap or Erlang's tracing
mechanism. For more details, please refer
link:http://hibari.github.com/hibari-doc/hibari-contributor-guide.en.html#_hibari_internal_tracepoints["Hibari internal tracepoints"]
section of Hibari Contributor's Guide.
