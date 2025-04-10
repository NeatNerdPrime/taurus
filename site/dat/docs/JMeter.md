# JMeter Executor

This executor type is used by default, it uses [Apache JMeter](https://jmeter.apache.org/) as underlying tool.

## JMeter Location & Auto-Installation

If there is no JMeter installed at the configured `path`, Taurus will attempt to install the latest JMeter and Plugins into
this location, by default `~/.bzt/jmeter-taurus/{version}/bin/jmeter`. You can change this setting to your preferred JMeter location (consider putting it into `~/.bzt-rc` file). All module settings that relates to JMeter path and auto-installing are listed below:
```yaml
modules:
  jmeter:
    path: ~/.bzt/jmeter-taurus/bin/jmeter
    download-link: https://archive.apache.org/dist/jmeter/binaries/apache-jmeter-{version}.zip
    version: 5.4.2  # minimal supported version of JMeter is 5.0
    force-ctg: true   # true by default
    detect-plugins: true
    fix-jars: true
    plugins:
    - jpgc-json=2.2
    - jmeter-ftp
    - jpgc-casutg
    plugins-manager:
      download-link: https://search.maven.org/remotecontent?filepath=kg/apc/jmeter-plugins-manager/{version}/jmeter-plugins-manager-{version}.jar
      version: 1.3   # minimum 0.16
    command-runner:
      download-link: https://search.maven.org/remotecontent?filepath=kg/apc/cmdrunner/{version}/cmdrunner-{version}.jar
      version: 2.2   # minimum 2.0
```
`force-ctg` allows you to switch off the usage of ConcurrentThreadGroup for jmx script modifications purpose. This group
provides `steps` execution parameter but requires `Custom Thread Groups` plugin (installed by default)

With `version` parameter you can ask for specific tool version or use autodetect with `auto` value. In that case
 taurus will analyze content of jmx file and try to guess appropriate the JMeter version.

`plugins` option lets you describe list of JMeter plugins you want to use. If `plugins` option isn't found only
following plugins will be installed: jpgc-casutg, jpgc-dummy, jpgc-ffw, jpgc-fifo, jpgc-functions, jpgc-json,
jpgc-perfmon, jpgc-prmctl, jpgc-tst. Keep in mind: you can change plugins list only for clean installation.
If you already have JMeter placed at `path` you need to remove it for plugins installation purpose.

`fix-log4j` provides hot fix for log4j vulnerabilities (CVE-2021-44228 & CVE-2021-45046) during jmeter installation.
This option requires internet connection for downloading from maven and may be turned off in proxy/intranet case.


[JMeter Plugins Manager](#https://jmeter-plugins.org/wiki/PluginsManager/) allows you to install necessary plugins for your jmx file automatically and this feature doesn't require clean installation. You can turn it off with `detect-plugins` option. If you use your own installation of JMeter (with `path` option) make sure it includes `jmeter-plugins-manager` 0.16 or newer.
If you want to change the URL from which the PluginsManager is downloaded or use a different version, configure `download-link` and/or `version` inside the `plugins-manager` block of the configuration.

For the PluginsManager to work correctly, you will also need the CommandRunner. It can also be downloaded automatically in the default version from the default location (as shown in the example above) or it can be configured similar to the PluginsManager.

## Run Existing JMX File
```yaml
execution:
- scenario: simple

scenarios:
  simple:
    script: tests/jmx/dummy.jmx
```

or simply `bzt tests/jmx/dummy.jmx`

In case when existing JMX file have multiple thread groups with different concurrency values while yaml config has its 
own concurrency, the main one in yaml will be divided in proportion to the concurrency values in thread groups. 
For example, if there are concurrency values 1 and 2 in two thread groups and 30 in yaml, the main will be divided in 
the ratio of 1:2, which means 10 to the first thread group and 20 to the second.

## JMeter Properties and Variables
There are two places to specify JMeter properties: global at module-level and local at scenario-level. Scenario properties are merged into global properties and resulting set comes as input for JMeter, see corresponding `.properties` file in artifacts.
You may also specify system properties for JMeter in system-properties section. They come as system.properties file in artifacts.

Global properties are set like this:

```yaml
modules:
  jmeter:
    properties:
      my-hostname: www.pre-test.com
      log_level.jmeter: WARN
      log_level.jmeter.threads: DEBUG
    system-properties:
      sun.net.http.allowRestrictedHeaders: "true"
```

Scenario-level properties are set like this:

```yaml
scenarios:
  prop_example:
    properties:
        my-hostname: www.prod.com
        log_level.jmeter: DEBUG
```
You can use properties in different parts of JMeter parameters to tune your script behaviour:
```yaml
execution:
- concurrency: ${__P(my_conc,3)}    # use `my_conc` prop or default=3 if property isn't found
  ramp-up: 30
  hold-for: ${__P(my_hold,10)}
  scenario: with_prop

modules:
  jmeter:
    properties:
      my_conc: 10
      my_hold: 20

scenarios:
  with_prop:
    requests:
    - http://blazedemo.com/${__P(sub_dir)}
    - https://blazemeter.com/${__P(sub_dir)}
    properties:
      my_hold: 15   # scenario-level property has priority
      sub_dir: contacts
```
Usage of variables are similar, but they can be used on scenario level only:
```yaml
scenarios:
  sc_with_vars:
    variables:
      subdir: contacts
      ref: http://gettaurus.org
    requests:
    - url: http://blazedemo.com/${subdir}
      headers:
        Referer: ${ref}
    - url: https://blazemeter.com/${subdir}
      headers:
        Referer: ${ref}
```

## Open JMeter GUI
When you want to verify or debug the JMX file that were generated from your requests scenario, you don't need to search for the file on disk, just enable GUI mode for JMeter module:

```yaml
modules:
  jmeter:
    gui: false  # set it to true to open JMeter GUI instead of running non-GUI test
```

For the command-line, use alias `-gui` or option `-o modules.jmeter.gui=true`, without the need to edit configuration file.

## Command-line Settings

You can specify special cli options for JMeter, for example:
```yaml
modules:
  jmeter:
    cmdline: --loglevel DEBUG
```

## Run JMeter in Distributed Mode
Distributed mode for JMeter is enabled with simple option `distributed` under execution settings, listing JMeter servers under it:

```yaml
execution:
- distributed:
  - host1.mynet.com
  - host2.mynet.com
  - host3.mynet.com
  scenario: some_scenario

scenarios:
  some_scenario:
    script: my-test.jmx
```
For accurate load calculation don't forget to choose different hostname values for slave hosts. If you have any properties specified in settings, they will be sent to remote nodes.

## Shutdown Delay
By default, Taurus tries to call graceful JMeter shutdown by using its UDP shutdown port (this works only for non-GUI). There is option to wait for JMeter to exit before killing it forcefully, called `shutdown-wait`. By default, its value is 5 seconds. Shutdown port number is searched automatically, starting from `shutdown-port` option value, by looking for unused ports.

## Modifications for Existing Scripts

JMeter executor allows you to apply some modifications to the JMX file before running JMeter (this affects both existing JMXes and generated from requests):

```yaml
scenarios:
  modification_example:
    script: tests/jmx/dummy.jmx
    modifications:
      disable:  # Names of the tree elements to disable
      - Thread Group 1
      enable:  # Names of the tree elements to enable
      - Thread Group 2
      set-prop:  # Set element properties, selected as [Element Name]>[property name]
        "HTTP Sampler>HTTPSampler.connect_timeout": "0"
        "HTTP Sampler>HTTPSampler.protocol": "https"
```
If selector for set-prop isn't found, taurus tries to create stringProp jmx element with last element of selector as name and sets given value for it. So you can create simple properties in jmx if it's necessary.

## Building Test Plan from Config

Scenario that has `requests` element makes Taurus to generate the script for underlying tools automatically. For now, this is available for JMeter and partially available for some other tools.

The `requests` element must contain a list of requests, each with its settings and child elements (assertions, extractors). Also, there are additional configuration elements for requests-based scenario, described below.

Scenario is the sequence of steps and some settings that will be used by underlying tools (JMeter, Gatling) on execution stage.

Scenarios are listed in top-level `scenarios` element and referred from executions by their alias:

```yaml
scenarios:
  get-requests:  # the alias for scenario
    requests:
    - http://localhost/1
    - http://localhost/2

execution:
- scenario: get-requests  # alias from above is used
```

### Global Settings

Scenario has some global settings:

```yaml
scenarios:
  get-requests:
    store-cache: true  # browser cache simulation, enabled by default
    store-cookie: true  # browser cookies simulation, enabled by default
    headers: # global headers
      header-name: header-value
    think-time: 1s500ms  # global delay between each request
    timeout: 500ms  #  timeout for connecting, receiving results, default value is 30s
    default-address: "https://www.blazedemo.com:8080"  # http request defaults scheme, domain, port
    keepalive: true  # true by default, applied on all requests in scenario
    retrieve-resources: true  # true by default, retrieves all embedded resources from HTML pages
    retrieve-resources-regex: ^((?!google|facebook).)*$  # regular expression used to match any resource
                                                         # URLs found in HTML document against. Unset by default
    concurrent-pool-size: 4  # concurrent pool size for resources download, 4 by default
    use-dns-cache-mgr: true  # use DNS Cache Manager to test resources
                             # behind dns load balancers. True by default.
    force-parent-sample: false  # generate only parent sample for transaction controllers.
                               # False by default
    content-encoding: utf-8  # global content encoding, applied to all requests.
                             # Unset by default
    follow-redirects: true  # follow redirects for all HTTP requests
    random-source-ip: false  # use one of host IPs to send requests, chosen randomly.
                             # False by default
    data-sources:  # these are data-sources options for Jmeter. See more info below.
    - path/to/my.csv  # this is a shorthand form
    - path: path/to/another.csv  # this is a full form
      delimiter: ';'
      quoted: false
      loop: true
      variable-names: id,name
      random-order: false
```
See more info about data-sources [here](DataSources.md).

If you want to use JMeter properties in `default-address`, you'll have to specify mandatory scheme and separate address/port. Like this: `default-address: https://${\__P(hostname)}:${\__P(port)}`.

It's possible to use follow specific values for choosing of `think-time`:
* 2s: constant value
* uniform(5s, 1s): random `think-time`, uniform distribution, possible values are 5±1 sec
* gaussian(1m, 1.5s): normal distribution of random values, where mean is 60s and deviation is 1.5s
* poisson(10s, 3s): poisson distribution, mean is 10s and range of values starts from 3s.

### Requests

Request objects can be of two kinds:
1. Plain HTTP requests
2. Logic blocks that allow user to control execution flow of test session.

#### HTTP Requests
The base element for requests scenario is HTTP Request. In its simplest form it contains just the URL as string:

```yaml
scenarios:
  get-requests:
    requests:
    - http://localhost/1
    - http://localhost/2
```

The full form for request is dictionary, all fields except `url` are optional:

```yaml
scenarios:
  my-req:
    requests:
    - url: http://blazedemo.com/  # url to hit
      method: GET  # request method (GET, POST, PUT, DELETE)
      label: homepage  # sampler label

      body: 'request-body-string'  # if present, will be used as body
      body:  # generate query string based on parameters and request type
        param1: value1
        param2: value2
      body-file: path/to/file.txt  # this file contents will be used as post body

      upload-files:  # attach files to form (and enable multipart/form-data)
      - param: summaryReport  # form parameter name
        path: report.pdf  # path to file
        mime-type: application/pdf  # optional, Taurus will attempt to guess it automatically

      headers:  # local headers that override global
        Authentication: Token 1234567890
        Referer: http://taurus.blazemeter/docs
      think-time: 1s  # local think-time, overrides global
      timeout: 1s  # local timeout, overrides global
      content-encoding: utf-8  # content encoding (at JMeter's level), unset by default
      follow-redirects: true  # follow HTTP redirects
      random-source-ip: false  # use one of host IPs to send the request (chosen randomly).
                               # False by default

      extract-regexp: {}  # explained below
      extract-jsonpath: {}  # explained below
      assert: []  # explained below
      jsr223: []  # explained below
```
Notes for `upload-files`:

- `POST` request method requires non-empty `param` values
- `PUT` method allows only one file in upload-files block

##### Extractors

Extractors are the objects that attached to request to take a piece of the response and use it in following requests.
The concept is based on JMeter's extractors. The following types of extractors are supported:

- by regular expression
- by boundary
- by JSONPath expression
- by CSS/JQuery selectors
- by XPath query

To specify extractors in shorthand form, use following configuration:

```yaml
scenarios:
  my-req:
    requests:
    - url: http://blazedemo.com/
      extract-regexp: # dictionary under it has form <var name>: <regular expression>
        page_title: <title>(\w+)</title>  #  must have at least one capture group
      extract-jsonpath: # dictionary under it has form <var name>: <JSONPath expression>
        varname: $.jsonpath[0].expression
    - url: http://blazedemo.com/${varname_1}/${page_title}  # that's how we use those variables
      extract-css-jquery: # dictionary under it has form <var name>: <CSS/JQuery selector>
        extractor1: input[name~=my_input]
    - url: http://blazedemo.com/${varname}/${extractor1}.xml
      extract-xpath:
        title: /html/head/title
```

Note that boundary extractor has no shorthand form. It can only be defined with full form.

The full form for extractors is:

```yaml
scenarios:
  my-req:
    requests:
    - url: http://blazedemo.com/
      extract-regexp:
        page_title:
          regexp: <title>(\w+)</title>  # regular expression
          default: NOT_FOUND  # default value to use when regexp not found
          match-no: 1  # if multiple values has matched, which match use (0=random)
          template: 1  # which capture group to take, integer or template string
          subject: body  #  subject for search
          scope: all  # check main and sub-samples
      extract-jsonpath:
        varname:
          jsonpath: $.jsonpath[0]  # jsonpath expression
          default: NOT_FOUND  # default value to use when jsonpath not found
          from-variable: JM_VAR # JMeter variable for search
          concat: false   # \
          scope: variable # - see below
          match-no: 4     # /
    - url: http://blazedemo.com/${varname}/${page_title}
      extract-css-jquery:
        extractor2:
          expression: input[name=JMeter]
          attribute: value
          match-no: 1
          default: NOT_FOUND
          scope: children   # check sub-samples
    - url: http://blazedemo.com/${varname}/${extractor2}.xml
      extract-xpath:
        destination:
          xpath: /order/client/address
          default: NOT_FOUND
          validate-xml: false
          ignore-whitespace: true
          match-no: -1
          use-namespaces: false
          use-tolerant-parser: false
    - url: http://blazedemo.com/${varname}.xml
      extract-boundary:
        pagetitle:
          subject: body  # extractor scope. values are: body, body-unescaped, body-as-document, response-headers, request-headers, url, code, message
          left: <title>  # left boundary to look for
          right: </title>  # right boundary to look for
          match-no: 1  # match number. 0 for random
          default: DEFVAL  # default value, if nothing is matched
```

You can choose `scope` for applying expressions. Possible value for targets are:
  - `all` - main sample and sub-samples
  - `children` - sub-samples
  - `variable` for search in JMeter variables
Default value of `scope` is empty, it means search in main sample only.

`match-no` allows to choose the specific result from several ones. Default value is 0 (random).
To get all values you can use `-1` - generation of variables _varname\_1_, _varname\_2_, etc.
It means if you ask for _some\_var\_name_ JMeter won't generate variable with exactly that name by default.

Possible subjects for regexp are:
  - `body`
  - `headers`
  - `http-code`
  - `url`

If several results are found they will be concatenated with ',' if `concat`.


##### Assertions

Assertions are attached to request elements and used to set fail status on the response. Fail status for the response is
not the same as response code for JMeter.
Currently, three types of response assertions are available.

First one checks http response fields, its short form looks like this:

```yaml
scenarios:
  my-req:
    requests:
    - url: http://blazedemo.com/
      assert:  # contains list of regular expressions to check
      - .+App.+
```

The full form has the following format:

```yaml
scenarios:
  my-req:
    requests:
    - url: http://blazedemo.com/
      assert:
       - contains:  # list of strings to check
         - .+App.+
         subject: body  # subject for search
         regexp: true  # treat string as regular expression
         not: false  # invert condition - fail if found
         assume-success: false  # mark sample successful before asserting it
```

Possible subjects are:
  - `body`
  - `headers`
  - `http-code`


The second assertion type is used to perform validation of JSON response against JSONPath expression.

```yaml
scenarios:
  my-req:
    requests:
    - url: http://blazedemo.com/
      assert-jsonpath:  # contains list of options
        - "$."  # if this JSONPath not found, assert will fail
        - "$.result[0]" # there can be multiple JSONPaths provided
```

Full form:

```yaml
scenarios:
  my-req:
    requests:
    - url: http://blazedemo.com/
      assert-jsonpath:
      - jsonpath: "$." # path to value, validation fails if path not exists
        validate: true # validate against expected value
        expected-value: "value" # the value we are expecting to validate, default: false
        regexp: true  # if the value is regular expression, default: true
        expect-null: false  # expected value is null
        invert: false # invert condition
```

And the third assertion type uses XPath query to validate XML response.

```yaml
scenarios:
  assertion-demo:
    requests:
    - url: http://blazedemo.com/
      assert-xpath:  # contains list of xpath queries
        - "/bookstore/book"  # if this XPath won't be matched, assert will fail
        - "/html/head/title" # you can provide multiple XPath queries
```

Full form:

```yaml
scenarios:
  my-req:
    requests:
    - url: http://blazedemo.com/
      assert-xpath:
      - xpath: "/html/head/title/text()='My title'" # query that compares XPath query result with some value
        use-tolerant-parser: false  # use error-tolerant XML parser
        ignore-whitespace: true # ignore whitespaces in XML (has no effect when `use-tolerant-parser` is true)
        validate: false # validate XML against its schema (has no effect when `use-tolerant-parser` is true)
        invert: false # invert condition
```

If sample is broken (RC isn't 200) and cause of assertion the same time, error message will be overwritten
with assertion message. Sometimes both of them are important and should be saved into results file.
For this case you can use following jmeter option:
```yaml
modules:
  jmeter:
    error-message-separator: ';'  # joins error and assert message if presented
```
##### JSR223 Blocks

Sometimes you may want to use a JSR223 Pre-/Post-Processor to execute a code block before or
after some requests. Taurus allows that with `jsr223` block. You can put this block into 
scenario level (block will run before/after each request in scenario) or into specific request. 

Minimal example that will generate one JSR223 Post Processor.
```yaml
scenarios:
  jsr-example:
    requests:
    - url: http://blazedemo.com/
      jsr223: 'vars.put("varname", "somevalue")'  # inline script to execute, unless script-file is specified
```

The example above uses defaults and inline script. If you want to use language different from groovy or use separate
script file, please use extended form of `jsr223` with key-value options.

Each jsr223 element can define the following fields:
- `language` - script language ('beanshell', 'bsh', 'ecmascript', 'groovy', 'java', 'javascript', 'jexl', 'jexl2')
- `script-file` - path to script file
- `script-text` - inline code, specified directly in config file
- `parameters` - string of parameters to pass to script, empty by default
- `execute` - whether to execute script before or after the request
- `compile-cache` - don't recompile scripts every time, turned on by default

If `execute` field is set to `after` - Taurus will generate a JSR223 PostProcessor, if set to `before` - a PreProcessor.
By default, it's set to `after`.

Long form:
```yaml
scenarios:
  jsr-example:
    requests:
    - url: http://blazedemo.com/
      jsr223:
      - language: javascript
        script-file: preproc.js
        parameters: foo bar
        execute: before
        compile-cache: false
      - language: beanshell
        script-file: postproc.bsh
        execute: after
```

#### Logic Blocks


Taurus allows to control execution flow with the following constructs:
- `if` blocks
- `once` blocks
- `loop` blocks
- `while` blocks
- `foreach` blocks
- `transaction` blocks
- `include-scenario` blocks
- `action` blocks

##### If Blocks

`if` blocks allow conditional execution of HTTP requests.

Each `if` block should contain a mandatory `then` field, and an optional `else` field. Both `then` and `else` fields
should contain lists of requests.

Here's a simple example:

```yaml
scenarios:
  if_example:
    variables:
      searchEngine: google
    requests:
    - if: '"${searchEngine}" == "google"'
      then:
        - https://google.com/
      else:
        - https://bing.com/
```

Note that Taurus compiles `if` blocks to JMeter's `If Controllers`, so `<condition>` must be in JMeter's format.


Logic blocks can also be nested:

```yaml
scenarios:
  nested_example:
    requests:
    - if: <condition1>
      then:
      - if: <condition2>
        then:
        - https://google.com/
        else:
        - https://yahoo.com/
      else:
      - https://bing.com/
```

And here's the real-world example of using `if` blocks:

```yaml
scenarios:
  complex:
    requests:
    # first request is a plain HTTP request that sets `status_code`
    # and `username` variables
    - url: https://api.example.com/v1/media/search
      extract-jsonpath:
        status_code: $.meta.code
        username: $.data.[0].user.username

    # branch on `status_code` value
    - if: '"${status_code}" == "200"'
      then:
        - https://example.com/${username}
```

##### Once blocks
`once` blocks is executed only once (per thread).
```yaml
scenarios:
  loop_example:
    requests:
    - once:
      - http://blazedemo.com/
```
They correspond to JMeter's `Once Only Controllers`.

##### Loop Blocks

`loop` blocks allow repeated execution of requests. Nested requests are to be specified with `do` field.

```yaml
scenarios:
  loop_example:
    requests:
    - loop: 10
      do:
      - http://blazedemo.com/
```

If you want to loop requests forever, you can specify string `forever` as `loop` value.
```yaml
scenarios:
  forever_example:
    requests:
    - loop: forever
      do:
      - http://blazedemo.com/
```

Note that `loop` blocks correspond to JMeter's `Loop Controllers`.

##### While Blocks

`while` block is similar to `while` loops in many programming languages. It allows conditional repeated execution of
requests. `while` blocks are compiled to JMeter's `While Controllers`.

```yaml
scenarios:
  while_example:
    requests:
    - while: ${JMeterThread.last_sample_ok}
      do:
      - http://blazedemo.com/
```

##### Foreach Blocks

`foreach` blocks allow you to iterate over a collection of values. They are compiled to JMeter `ForEach Controllers`.

Syntax:
```yaml
requests:
- foreach: <elementName> in <collection>
  do:
  - http://${elementName}/
```

Concrete example:
```yaml
scenarios:
  complex_foreach:
    requests:
    - url: https://api.example.com/v1/media/search
      extract-jsonpath:
        usernames: $.data.[:100].user.username  # grab first 100 usernames
    - foreach: name in usernames
      do:
      - https://example.com/user/${name}
```

##### Transaction Blocks

`transaction` blocks allow wrapping http requests in a transaction. `transaction` blocks correspond to JMeter's
`Transaction Controllers`.

Example:
```yaml
scenarios:
  transaction_example:
    requests:
    - transaction: Customer Session
      force-parent-sample: false  # False by default
      include-timers: true  # add timers and pre-/post-processors execution time to samples
      do:
      - http://example.com/shop
      - http://example.com/shop/items/1
      - http://example.com/shop/items/2
      - http://example.com/card
      - http://example.com/checkout
```
Take note: you can specify force-parent-sample on both levels - scenario and transaction. If both are found local (transaction) value has priority.

##### Include Scenario Blocks
`include-scenario` block allows you to include scenario into another one. You can use it to split your test plan into
a few of independent scenarios that can be reused.

Example:
```yaml
scenarios:
  login:
    data-sources:
    - logins.csv
    requests:
    - url: http://example.com/login
      method: POST
      body:
        user: ${username}
        password: ${password}
  logout:
    requests:
    - url: http://example.com/logout
      method: POST
  shop-session:
    requests:
    - include-scenario: login
    - http://example.com/shop/items/1
    - http://example.com/checkout
    - include-scenario: logout
```

Taurus translates each `include-scenario` block to a JMeter's `Simple Controller` and puts all scenario-level
settings and requests there.
Keep in mind: the following scenario-level parameters of including scenario have no effect for included ones:
- `keepalive`
- `timeout`
- `think-time`
- `follow-redirects`
- `properties`

##### Action Blocks

`action` block allows you to specify a thread-specific action that will be performed. You can use it to pause or stop
the current thread, or force it to go to the next loop iteration.

The following actions are available:
- `pause` - pause the target thread (pause duration is controlled with `pause-duration` field)
- `stop` - stop the target thread gracefully
- `stop-now` - stop the test without waiting for samples to complete
- `continue` - send target thread to the next iteration of the loop

Actions can be applied to the following targets:
- `current-thread` - set by default
- `all-threads` - action will be applied to all threads (unavailable for `continue` action)

Examples:
```yaml
scenarios:
  action_example:
    requests:
    - action: pause
      target: current-thread
      pause-duration: 1s500ms
    - action: stop-now
      target: all-threads
```

##### Set Variables Blocks

`set-variables` block allows you to set JMeter variables from other variables.

Example:
```yaml
scenarios:
  set_vars_example:
    variables:
      foo: BAR
    requests:
    - http://blazedemo.com/?foo=${foo}
    - set-variables:
        foo: BAZ
```

This example will set initial value of `${foo}` to be "BAR", but after first iteration it will be
changed to "BAZ".

#### HTTP Authorization
See [RFC2617](https://datatracker.ietf.org/doc/html/rfc2617) for http authorization details

You can use three follow forms for such purposes:
```yaml
scenarios:
  simply:
    authorization:
      url: auth_server_addr
      name: my_username
      password: my_pass

```
It's the shortest form for quick setup. You can use several authorizations:

```yaml
scenarios:
  multi_auth:
    authorization:
    - url: auth_server_addr1
      name: username1
      password: pass1
    - url: auth_server_addr2
      name: username2
      password: pass2
```
If you want to reset authorization for each test iteration you have to use `clear` flag and full form:
```yaml
scenarios:
  full_auth:
    authorization:
      clear: true   # false by default
      list:
      - url: auth_server_addr1
        name: username1
        password: pass1
      - url: auth_server_addr2
        name: username2
        password: pass2
```
Possible authorization params and their value are:
* url: link to the resource you want to access
* username & password: your credential
* domain: non-standard parameter, can be used instead of url
* realm: protected space
* mechanism: digest (default) or kerberos.
Required of them are username & password and one of url & domain.
For implementation of authorization Taurus uses JMeter HTTP Authorization Manager.

### Client Certificate Based Authorization
Taurus allows the use of client certificate based authorization using JMeter executor.

There are generally two scenarios for client certificate based authentication.

#### If you require only one certificate for your whole test:
* For this method, you can use either a certificate type of pkcs12 (*.p12) or Java Key Store (JKS)
* Set the certificate path and the certificate password in JMeter system-properties section
```yaml
modules:
  jmeter:
    system-properties:
      javax.net.ssl.keyStore: ${BASE_DIR}/test-data/my-client-certificates.p12
      javax.net.ssl.keyStorePassword: MyClientCertificatePassword
```
* If the certificate has more than one certificate in it, JMeter will use only one
* Keep an eye on the jmeter.log file to confirm if the certificate loaded correctly
```
INFO o.a.j.u.SSLManager: Total of 1 aliases loaded OK from keystore
```

#### If you require multiple client certificates for your test (e.g. one per user session):
* For this method, you need a single JKS file with all the certificates added with their own aliases
    * You can use java's keytool utility to convert other formats to JKS and add it to single JKS file with aliases
* Create a CSV file with list of corresponding certificate aliases that are required for your test
* Add a data-source for this CSV file for taurus to load
* Use keystore-config to set various parameters about the keystore as shown below
    * For each iteration of execution, an alias from CSV file will be read and stored in the variable mentioned
    * And the corresponding certificate for the alias will be used when setting up the https connection
```yaml
scenarios:
  client-cert-scenario:
    data-sources: # Read cert aliases from this CSV file
    - path: ${BASE_DIR}/test-data/my-client-certificate-aliases.csv
      delimiter: ','
      quoted: false  # allow quoted data
      loop: true  # loop over in case of end-of-file reached if true, stop thread if false
      variable-names: certalias  # variable-name here needs to be used in keystore-config next
    keystore-config:
      variable-name: certalias # variable name used in data-source element
      start-index: 0 # The index of the first key to use in Keystore, 0-based.
      end-index: 99 # The index of the last key to use in Keystore, 0-based.
      preload: true
```
* Also add the JKS file and its password in the system-properties section
```yaml
modules:
  jmeter:
    properties:
    system-properties:
      javax.net.ssl.keyStore: ${BASE_DIR}/test-data/my-client-certificates-in-one-jks.jks
      javax.net.ssl.keyStorePassword: MyClientCertificateJKSFilePassword
```
* Keep an eye on the jmeter.log file to confirm if the all aliases loaded correctly
```
INFO o.a.j.c.KeystoreConfig: Configuring Keystore with (preload: 'True', startIndex: 0, endIndex: 99, clientCertAliasVarName: 'certalias')
INFO o.a.j.u.JsseSSLManager: Using default SSL protocol: TLS
INFO o.a.j.u.JsseSSLManager: SSL session context: per-thread
INFO o.a.j.u.SSLManager: JmeterKeyStore Location: /tmp/test-data/my-client-certificates-in-one-jks.jks type JKS
INFO o.a.j.u.SSLManager: KeyStore created OK
INFO o.a.j.u.SSLManager: Total of 100 aliases loaded OK from keystore
```

## User cookies
Taurus allows you to set up some user cookies with follow syntax:
```yaml
scenarios:
  little_cookies:
    requests:
    - http://blazedemo.com/login
    - https://blazemeter.com/pricing
    cookies:
    - name: n0  # name of cookie, required
      value: v0 # value of cookie, required
      domain: blazedemo.com # hosts to which the cookie will be sent, required
    - name: n1
      value: v1
      domain: blazemeter.com
      path: /pricing  # must exist in the request for sending cookie
      secure: true    # send only through https, optional, default: false
```
## JMeter Test Log
You can tune JTL file content with option `write-xml-jtl`. Possible values are 'error' (default), 'full', or any other value for 'none'. Keep in mind: max `full` logging can seriously load your system.
```yaml
execution:
- write-xml-jtl: full
  scenario: simple_script

scenarios:
  simple_script:
    script: my.jmx

```
Another way to adjust verbosity is to change flags in `xml-jtl-flags` dictionary. Next example shows all flags with default values (you don't have to use full dictionary if you want to change some from them):
```yaml
modules:
  jmeter:
    xml-jtl-flags:
      xml: true
      fieldNames: true
      time: true
      timestamp: true
      latency: true
      connectTime: true
      success: true
      label: true
      code: true
      message: true
      threadName: true
      dataType: true
      encoding: true
      assertions: true
      subresults: true
      responseData: false
      samplerData: false
      responseHeaders: true
      requestHeaders: true
      responseDataOnError: true
      saveAssertionResultsFailureMessage: true
      bytes: true
      threadCounts: true
      url: true
```

Remember: some logging information might be used by `[assertions](#Assertions)` so change log verbosity can affect them.

### CSV file content configuration

By default, the kpi.jtl file does not have enough fields for JMeter's HTML dashboard generator to operate.   
You can change this with option `csv-jtl-flags` like this:

```yaml
modules:
  jmeter:
    csv-jtl-flags:
      saveAssertionResultsFailureMessage: true
      sentBytes: true

services:
- module: shellexec
  post-process:
  - '[ -f "${TAURUS_ARTIFACTS_DIR}/kpi.jtl" ] && /path/to/jmeter -g "${TAURUS_ARTIFACTS_DIR}/kpi.jtl" -o "${TAURUS_ARTIFACTS_DIR}/dashboard" -j "${TAURUS_ARTIFACTS_DIR}/generate_report.log" '
```

Next example shows all the default flags with default values (you don't have to use full dictionary if you want to change some from them):

```yaml
modules:
  jmeter:
    csv-jtl-flags:
      xml: false
      fieldNames: true
      time: true
      timestamp: true
      latency: true
      connectTime: true
      success: true
      label: true
      code: true
      message: true
      threadName: true
      dataType: false
      encoding: false
      assertions: false
      subresults: false
      responseData: false
      samplerData: false
      responseHeaders: false
      requestHeaders: false
      responseDataOnError: false
      saveAssertionResultsFailureMessage: false
      bytes: true
      hostname: true
      threadCounts: true
      url: false
```

You can also add flags which are not listed above, as long as JMeter recognizes them:

```yaml
modules:
  jmeter:
    csv-jtl-flags:
      sentBytes: true
      idleTime: true
```

Note: JMeter dashboard generator **requires** the following settings:

```yaml
modules:
  jmeter:
    csv-jtl-flags:
      time: true
      timestamp: true
      latency: true
      connectTime: true
      success: true
      label: true
      code: true
      message: true
      threadName: true
      saveAssertionResultsFailureMessage: true
      bytes: true
      threadCounts: true
      sentBytes: true  # JMeter 4.0 or above
```

## JMeter JVM Memory Limit

You can tweak JMeter's memory limit (aka, `-Xmx` JVM option) with `memory-xmx` setting.
Use `K`, `M` or `G` suffixes to specify memory limit in kilobytes, megabytes or gigabytes.

Example:
```yaml
modules:
  jmeter:
    memory-xmx: 4G  # allow JMeter to use up to 4G of memory
```

## Protocol Handlers

JMX generator used by Taurus supports extensibility. It can be controlled with `protocol` option.

`protocol` setting at request level specifies which protocol handler to use to generate corresponding JMX.
`protocol` can also be specified at scenario level, which will apply it to all requests in the scenario.

```yaml
modules:
  jmeter:
    protocol-handlers:  # list of protocols supported by JMX generator
      http: bzt.jmx.http.HTTPProtocolHandler
    default-protocol: http  # default protocol used by JMX generator

scenarios:
  protocols-demo:
    protocol: http  # default protocol
    requests:
    - url: http://blazedemo.com/

execution:
- executor: jmeter
  scenario: protocols-demo
```

## MQTT Protocol Load Testing

JMeter tool can be used for load testing of mqtt infrastructure. This protocol is very popular in IoT world.
Usually the target of test is mqtt server ('broker'). Taurus can emulate messages of sensors and actuators to check 
whether this broker is good enough for specific load.

```yaml
execution:
- concurrency: 3
  hold-for: 5s
  scenario: sc_pub
- concurrency: 1
  scenario: sc_sub

scenarios:
  sc_pub:
    protocol: mqtt
    requests:
    - cmd: connect # first step for every mqtt client
      addr: 127.0.0.1
    - cmd: publish  # publishing message into topic
      topic: t1 # topic name
      message: my_message # content of message
    - cmd: disconnect # last step for every mqtt client

  sc_sub:
    protocol: mqtt
    requests:
    - cmd: connect
      addr: 127.0.0.1
    - cmd: subscribe  # subscribing on topic
      topic: t1
      time: 3s        # check topic for 3 seconds
      min-count: 20   # at least 20 messages must be received
    - cmd: disconnect
```
Here you see three sensors ('publisher') and one checker ('subscriber'). Messages that created by sensors
must be handled by broker and redirected to appropriate subscribers.
All logic blocks, data sources and many other functionality of JMeter Executor are available with mqtt protocol as well.


## gRPC Protocol Load Testing

JMeter tool can also perform load testing of gRPC services, via [jmeter-grpc-request](https://github.com/zalopay-oss/jmeter-grpc-request) plugin.

Here's a very basic request (the `url` and `grpc-proto-folder` fields are required):

```yaml
# Install the relevant JMeter plugin
modules:
  jmeter:
    plugins:
      - jmeter-grpc-request

scenarios:
  hello-world:
    protocol: grpc
    grpc-proto-folder: /path/to/protobuf/folder
    requests:
    - url: http://api.example.com:8888/com.example.HelloWorldService/sayHello
```

Here's the full set of options:

```yaml
scenarios:
  my-req:
    protocol: grpc
    # These options can be set at the scenario level, or at the request level
    timeout: 5s
    grpc-proto-folder: /path/to/protobuf/folder
    grpc-lib-folder: /path/to/grpc/lib/folder
    grpc-max-inbound-message-size: 4194304
    grpc-max-inbound-metadata-size: 8192
    tls-disable-verification: false
      
    requests:
      # The URL is parsed into (scheme, hostname, port, path) which are propagated to the jmeter-grpc-request plugin
      # The "path" component of the URL is used as the "Full Method" param
      # If the scheme is "https" then the SSL/TLS param will be set to "true"
    - url: http://api.example.com:8888/com.example.HelloWorldService/sayHello  # url to hit
      label: homepage  # sampler label, defaults to the value of url

      # Request body can be sepcified as a JSON string, as a dictionary, or as JSON in a separate file
      body: '{ "param1": "value1", "param2": value2 }'  # if present, will be used as body
      body:  # nested attributes will be converted to JSON
        param1: value1
        param2: value2
      body-file: path/to/file.json  # this file contents will be used as request body

      # Request metadata can be sepcified as a JSON string or as a dictionary
      metadata: 'key1:value1,key2:value2'
      metadata: '{"key1":"Value1", "key2":"value2"}'
      metadata:
        key1: Value1
        key2: value2

      # These options can be set at the scenario level, or at the request level
      timeout: 5s  # Timeout is used for both "deadline" and "channelAwaitTermination"
      grpc-proto-folder: /path/to/protobuf/folder
      grpc-lib-folder: /path/to/grpc/lib/folder
      grpc-max-inbound-message-size: 4194304
      grpc-max-inbound-metadata-size: 8192
      tls-disable-verification: false
      
      extract-regexp: {}  # same as described for HTTP sampler above
      extract-jsonpath: {}  # same as described for HTTP sampler above
      assert: []  # same as described for HTTP sampler above
      jsr223: []  # same as described for HTTP sampler above
```

All logic blocks, data sources and many other functionality of JMeter Executor are available with grpc protocol as well.
