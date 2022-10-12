# SparrowCI

SparrowCI - super fun and flexible CI system with many programming languages support.

# Quick Start

Install SparrowCI

```bash
# choose any convenient package manager: 
sudo apk add sparrowci
sudo apt-get install sparrowci
sudo yum install sparrowci
```

Create build scenario `sparrow.yaml`:

```yaml
  tasks:
    -
      name: make_task
      language: Bash
      config:
        prefix: ~/
        with_test: false
      code: |
        prefix=$(config prefix)
        make
        if test $(config with_test) = "true"; then
          make test
        fi
        sudo make install --prefix=$prefix
        echo "Hello from Bash"
    -
      name: raku_task
      language: Raku
      default: true # start scenario with that task
      depends: 
        - 
          name: make_task
          config:
            with_test: true
      cleanup:
        - 
            name: python_task
            config:
              foo: baz
      code: |
        say "Hello from Raku";
        update_state %( message => "OK" );
    -
      name: python_task
      language: Python
      config:
        foo: bar # default value
      code: |
        state = config()['tasks']['raku_task']
        print("Hello from Python")
        
        print(f"I can read output data from other tasks: {state['message']}")
        print(f"named parameter: {config()['foo']}")
```

This example scenario would execute Bash task and Python task and then 
execute Raku task. Task dependencies are just DAG and ensured by `depends`/`cleanup`
sections. 

Task that is marked as `default: true` is executed first when a 
scenario is triggered.

To execute scenario add it to your git repo and assign tasks to SparrowCI service:


```bash
git add sparrow.yaml
git commit -a -m "my sparrowci scenario"
git push

sparrow_ci login # login into SparrowCI account 
sparrow_ci register # register your git project, this will trigger a new build soon

```

# Advanced topics

Consider more sophisticated example:

```yaml
  tasks:
    -
      name: make_file
      language: Bash
      code: |
        mkdir -p foo/bar
        touch foo/bar/file.txt
      artifacts:
        out:
          -
            name: file.txt
            path: foo/bar/file.txt
    - name: parser
      language: Ruby
      default: true
      depends:
      -
        name: make_file
      code:
        File.readlines('.artifacts/file.txt').each do |line|
          puts(line)
        end
      artifacts:
        in:
          - file.txt 
```

## Artifacts

Tasks might produce artifacts that become visible within other tasks.

In this example, `parser` task would wait till `make_file` is executed and produce artifact
for file located at `foo/bar/file.txt`, the artifact will be registered and copied into 
the central SparrowCI system with  unique name `file.txt`.

The artifact then will be ready to be consumed for other tasks. The local file
representing consumed artifact will be located at `.artifacts` folder.

## Sparrow Plugins

Sparrow plugins are reusable tasks one can run from within their scenarios:

```yaml
  tasks:
    -
      name: dpl_check
      plugin: k8s-deployment-check
        config:
          name: animals
          namespace:  pets
          image:  blackcat:1.0.0
          command: /usr/bin/cat
          args:
            - eat
            - milk
            - fish
          env:
            - ENABLE_LOGGING
          volume-mounts:
            foo-bar: /opt/foo/bar

```

In this example plugin "k8s-deployment-check" checks k8s deployment resource.

Available plugins are listed on SparrowHub repository - https://sparrowhub.io

## Tasks

Tasks are elementary build units. They could be written on various languages
or executed as external Sparrow plugins.

Every task is executed in separated environment and does not see other tasks.

Artifacts and tasks output data is mechanism how tasks communicate with each other.

## Workers

Workers execute tasks.

SparrowCI workers are ephemeral docker alpine instances, that are created
for every build and then destroyed. If you need more OS support please
let me know.

## Parallel tasks execution

Tasks could be executed on parallel workers simultaneously for efficiency.

Documentation - TDB

## Dependencies

Tasks dependencies are implemented via `depends`/`cleanup` tasks lists.

`depends`/`cleanup` sections allows to execute tasks before/after a given one.

`cleanup` tasks executed unconditional on main tasks status (success/failure).

if any of  `depends` tasks fail the main, dependent task is not executed and the whole
scenario terminated.

`depends`/`cleanup` tasks are executed in no particular order and 
_potentially_ on separated hosts.


## Subtasks

Subtasks allows to split a big task on small pieces (sub tasks) and
call them as functions:

```yaml
- tasks:
  -
    name: app_test
    language: Raku
    main: |
      for ('http://raku.org', 'https://raku.org') -> $url {
        run_task http_check, %(
          url => $url
        )
      }
    subtasks:
      -
        name http_check
        language: Bash
        code: |
          curl -f $url
    code: |
      say "finshed"
```

## Using Programming Languages 

Currently SparrowCI supports following list of programming languages:

- Raku
- Bash
- Python
- Perl
- Ruby
- Powershell

To chose a language for underling task just use `language: $language` statement:

```yaml
  name: powershell_task
  language: Powershell
  code: |
    Write-Host "Hello from Powershell"
```

## Tasks parameters and configuration

Every task might have some default configuration:

```yaml
  name: pwsh_task
  language: Powershell
  config:
    message: Hello
  code: |
    echo "you've said:" $(config message)
```

Default configuration parameters could be overridden by tasks parameters:

```yaml
  tasks:
    -
      name: main_task
      language: Bash
      default: true
      depends: 
        - 
          name: pwsh_task
          config:
            message: "How are you?"
```

## Task output data

Task might have output data that is later becomes available
within _dependent_ task that run this task as dependency:

```yaml
  tasks:
    -
      name: parser
      language: Ruby
      default: true
      name: ruby_task
      code: |
        update_state(Hash["message", "I code in Ruby"])
    -
      name: raku_task
      language: Raku
      default: true
      depends: 
        - 
          name: ruby_task
      code: |
        say "Hello from Raku";
        my $ruby_task_message = config()<tasks><ruby_task><message>;
```

`update_state()` function accepts HashMap as parameter and available for all programming languages, excepts Bash.

`config()` function is used to access a task output data.

If the same task executed as a dependency, use task local name to tell output data from
different runs of the same task:

```yaml
  tasks:
    -
      name: parser
      language: Ruby
      default: true
      name: ruby_task
      code: |
        rand_string = (0...8).map { (65 + rand(26)).chr }.join
        update_state(Hash["random_message", rand_string])
    -
      name: raku_task
      language: Python
      default: true
      depends: 
        - 
          name: python_task
          localname: ruby_task1
        - 
          name: ruby_task
          localname: ruby_task2
      code: |
        print("Hello from Python")
        ruby_task1_message = config()['tasks']['ruby_task1']['random_message'];
        ruby_task2_message = config()['tasks']['ruby_task2']['random_message'];
```

## Plugins parameters and output data

Plugins have default and input parameters, as well as states (output data).

## Task checks

Task checks allow to create rules to verify scripts output, it useful when creating
Bash scripts in quick and dirty way, where there is no convenient way to validate scripts logic:

```yaml
tasks:
  -
    name: create-database
    language: Bash
    code: |
      mysql -c "create database foo" -h 127.0.0.1 2>&1
    check: |
      database .* exists | created   
```

Check rules are based on Raku regular expression, follow this link - https://github.com/melezhik/Sparrow6/blob/master/documentation/taskchecks.md
to know more

## Source code and triggering

Build triggering happens automatically upon any changes in a source code.

The source code is checked out into `./source` local folder.


# Examples

Here is just short list of some possible scenarios.

## Make build

```yaml
  tasks:
    -  
      name: build
      language: Bash
      default: true
      config:
        url: https://github.com/rakudo/rakudo.git
      code: |
        set -e
        git clone $(config url) scm
        cd scm
        perl Configure.pl --gen-moar --gen-nqp --backends=moar
        make
        make install
```

## Python build

```yaml
tasks:
  -
    name: install_python
    # install Python dependencies
    plugin: sparkyci-package-python
  - 
    name: unit tests
    language: Bash
    depends:
      -
        name: install_python
    code: |
      set -e
      cd source/
      sudo pip3 install -r requirements.txt
      pytest
```

## Raku build

TBD

## Golang build

TBD


