# Logparser Input Plugin

The `logparser` plugin streams and parses the given logfiles. Currently it
has the capability of parsing "grok" patterns from logfiles, which also supports
regex patterns.

### Configuration:

```toml
[[inputs.logparser]]
  ## Log files to parse.
  ## These accept standard unix glob matching rules, but with the addition of
  ## ** as a "super asterisk". ie:
  ##   /var/log/**.log     -> recursively find all .log files in /var/log
  ##   /var/log/*/*.log    -> find all .log files with a parent dir in /var/log
  ##   /var/log/apache.log -> only tail the apache log file
  files = ["/var/log/apache/access.log"]
  ## Read file from beginning.
  from_beginning = false

  ## Parse logstash-style "grok" patterns:
  ##   Telegraf built-in parsing patterns: https://goo.gl/dkay10
  [inputs.logparser.grok]
    ## This is a list of patterns to check the given log file(s) for.
    ## Note that adding patterns here increases processing time. The most
    ## efficient configuration is to have one pattern per logparser.
    ## Other common built-in patterns are:
    ##   %{COMMON_LOG_FORMAT}   (plain apache & nginx access logs)
    ##   %{COMBINED_LOG_FORMAT} (access logs + referrer & agent)
    patterns = ["%{COMBINED_LOG_FORMAT}"]
    ## Name of the outputted measurement name.
    measurement = "apache_access_log"
    ## Full path(s) to custom pattern files.
    custom_pattern_files = []
    ## Custom patterns can also be defined here. Put one pattern per line.
    custom_patterns = '''
    '''
```

### Grok Parser

The grok parser uses a slightly modified version of logstash "grok" patterns,
with the format

```
%{<capture_syntax>[:<semantic_name>][:<modifier>]}
```

Telegraf has many of it's own
[built-in patterns](./grok/patterns/influx-patterns),
as well as supporting
[logstash's builtin patterns](https://github.com/logstash-plugins/logstash-patterns-core/blob/master/patterns/grok-patterns).


The best way to get acquainted with grok patterns is to read the logstash docs,
which are available here:
  https://www.elastic.co/guide/en/logstash/current/plugins-filters-grok.html


If you need help building patterns to match your logs,
you will find the https://grokdebug.herokuapp.com application quite useful!


By default all named captures are converted into string fields.
Modifiers can be used to convert captures to other types or tags.
Timestamp modifiers can be used to convert captures to the timestamp of the
parsed metric.  If no timestamp is parsed the metric will created using the
current time.


- Available modifiers:
  - string   (default if nothing is specified)
  - int
  - float
  - duration (ie, 5.23ms gets converted to int nanoseconds)
  - tag      (converts the field into a tag)
  - drop     (drops the field completely)
- Timestamp modifiers:
  - ts               (This will auto-learn the timestamp format)
  - ts-ansic         ("Mon Jan _2 15:04:05 2006")
  - ts-unix          ("Mon Jan _2 15:04:05 MST 2006")
  - ts-ruby          ("Mon Jan 02 15:04:05 -0700 2006")
  - ts-rfc822        ("02 Jan 06 15:04 MST")
  - ts-rfc822z       ("02 Jan 06 15:04 -0700")
  - ts-rfc850        ("Monday, 02-Jan-06 15:04:05 MST")
  - ts-rfc1123       ("Mon, 02 Jan 2006 15:04:05 MST")
  - ts-rfc1123z      ("Mon, 02 Jan 2006 15:04:05 -0700")
  - ts-rfc3339       ("2006-01-02T15:04:05Z07:00")
  - ts-rfc3339nano   ("2006-01-02T15:04:05.999999999Z07:00")
  - ts-httpd         ("02/Jan/2006:15:04:05 -0700")
  - ts-epoch         (seconds since unix epoch)
  - ts-epochnano     (nanoseconds since unix epoch)
  - ts-"CUSTOM"


CUSTOM time layouts must be within quotes and be the representation of the
"reference time", which is `Mon Jan 2 15:04:05 -0700 MST 2006`
See https://golang.org/pkg/time/#Parse for more details.

#### Timestamp Examples

This example input and config parses a file with a custom timestamp convertion:

```
2017-02-21 13:10:34 value=42
```

```toml
[[inputs.logparser]]
  [inputs.logparser.grok]
    patterns = ['%{TIMESTAMP_ISO8601:timestamp:ts-"2006-01-02 15:04:05"} value=%{NUMBER:value:int}']
```

This example parses a file using a built-in conversion and a custom pattern:

```
Wed Apr 12 13:10:34 PST 2017 value=42
```

```toml
[[inputs.logparser]]
  [inputs.logparser.grok]
	patterns = ["%{TS_UNIX:timestamp:ts-unix} value=%{NUMBER:value:int}"]
    custom_patterns = '''
      TS_UNIX %{DAY} %{MONTH} %{MONTHDAY} %{HOUR}:%{MINUTE}:%{SECOND} %{TZ} %{YEAR}
    '''
```

#### TOML Escaping

When saving patterns to the configuration file, keep in mind the different TOML
[string](https://github.com/toml-lang/toml#string) types and the escaping
rules for each.  These escaping rules must be applied in addition to the
escaping required by the grok syntax.  Using the Multi-line line literal
syntax with `'''` may be useful.

The following config examples will parse this input file:

```
|42|\uD83D\uDC2F|'telegraf'|
```

Since `|` is a special character in the grok langauge, we must escape it to
get a literal `|`.  With a basic TOML string, special characters such as
backslash must be escaped, requiring us to escape the backslash a second time.

```toml
[[inputs.logparser]]
  [inputs.logparser.grok]
    patterns = ["\\|%{NUMBER:value:int}\\|%{UNICODE_ESCAPE:escape}\\|'%{WORD:name}'\\|"]
    custom_patterns = "UNICODE_ESCAPE (?:\\\\u[0-9A-F]{4})+"
```

We cannot use a literal TOML string for the pattern, because we cannot match a
`'` within it.  However, it works well for the custom pattern.
```toml
[[inputs.logparser]]
  [inputs.logparser.grok]
    patterns = ["\\|%{NUMBER:value:int}\\|%{UNICODE_ESCAPE:escape}\\|'%{WORD:name}'\\|"]
    custom_patterns = 'UNICODE_ESCAPE (?:\\u[0-9A-F]{4})+'
```

A multi-line literal string allows us to encode the pattern:
```
[[inputs.logparser]]
  [inputs.logparser.grok]
    patterns = ['''
	  \|%{NUMBER:value:int}\|%{UNICODE_ESCAPE:escape}\|'%{WORD:name}'\|
	''']
    custom_patterns = 'UNICODE_ESCAPE (?:\\u[0-9A-F]{4})+'
```

#### Additional Information

- https://www.influxdata.com/telegraf-correlate-log-metrics-data-performance-bottlenecks/
