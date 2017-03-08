hobson
======

A software build tool with emphasis on reproducibility, offline use, speed, and an absence of magic. Hobson is currently in the design phase.

#### Motivation

While using Maven for a moderately-sized software project, I found myself annoyed by some of its choices. Non-trivial use seemed to require Maven plug-ins, and by default on every build it would check for and download new versions of those plug-ins, with no particular guarantee of backward compatibility. It also wanted to reevaluate library dependencies on every build, and download new versions of them, and by default it would build "site" documentation and other artifacts of limited value. Changing the defaults meant wading through a long "effective POM" file to find the relevant settings.

Both Ant and Maven include significant improvements over the venerable Unix Make utility, but lost some of Make's inherent simplicity. They are also Java-specific. The goal for Hobson is to be no less modern than Ant and Maven, while restoring the simplicity of Make. Hobson will do what you tell it, while leaving you in control. It will have built-in tasks (like Make does) but these tasks will be the same as what you could write yourself, not Java code you can't see.

#### Overview

Superficially, Hobson will resemble Ant. At startup time, Hobson will read an XML configuration file and optional properties files; based on those files it will determine what order to execute compilers or other tools, and then execute them, noting which ones succeeded and failed.

Digging deeper, Hobson will more closely resemble Make. The build steps are defined as commands to be executed in order to update an output from one or more inputs, and Hobson will perform a topological sort to determine what order to run the commands. Unlike Make, where the inputs and outputs are individual files, Hobson will use Ant-style filesets as inputs and outputs.

#### Example

The following example uses the Maven source layout for Java for familiarity.

    <?xml version="1.0">
    <project default="jar" version="0.1">
        <source name="src" pattern="src/**/*.java"/>
        <target name="bin" pattern="bin/**/*.class"/>
        <target name="jar" pattern="hello.jar"/>

        <step depends="src" produces="bin">
            <run task="javac"/>
        </step>

        <step depends="bin" produces="jar">
            <run task="jar"/>
        </step>
    </project>

The version number is the version of Hobson you are using, or the version you want a later version of Hobson to emulate. The "depends" and "produces" attributes are comma-separated lists of names of sources or targets. The tasks can either be defined in the build file or taken from Hobson's built-ins:

    <task name="javac">
        <mkdirs>${output.dir}</mkdirs>
        <cmd>${javac} ${javac.options} -d ${output.dir}
            <path>-sourcepath ${input.dir}</path>
            <opt>-s ${javac.generated.source.dir}</opt>
            ${input.filename}
        </cmd>
    </task>

When a &lt;step&gt; runs a &lt;task&gt;, Hobson insures that filesets named ${input} and ${output} are defined for the &lt;task&gt;. These can be set explicitly, for example:

    <run task="javac" input="${src}"/>

If not set explicitly, the defaults are simply

    input="${${depends}}"
    output="${${produces}}"

#### Property files

Syntax for property files will be similar to Java property files, but simpler. Each line has one of the two formats:

    key=value
    key:values

Unlike in Java, the file encoding is always UTF-8, the key can't contain any ASCII whitespace or punctuation except period and underscore, the equal sign or colon is required, and leading and trailing whitespace is ignored. The backslash has no special meaning (thus no \u Unicode or other escape sequences.) In expressions, ${key} will be either a single string or a list of strings, depending on whether an equal sign or colon was used. For example:

    javac=C:\Program Files\Java\jdk1.8.0\bin\javac.exe
    javac.options:-g -Xlint

#### Expressions

Hobson will use the ${} notation for expressions. The value of an expression can be

* a single string
* a list of zero or more strings
* a fileset
* a multi-fileset

String-valued expressions can be defined in the build file or a property file. Filesets can only be defined in the build file. Example expressions:

    ${}              empty
    ${$}             literal dollar sign
    ${/} or ${\}     platform specific filename separator
    ${:} or ${;}     platform specific path separator

    ${a,b}           multi-fileset containing the files included in ${a} and ${b}
    ${a?x:y}         ${x} if ${a} is non-empty, otherwise ${y}
    ${a?x}, ${a?:y}  (analogous)
    ${a??y}          ${a} if non-empty, otherwise ${y}

    ${a/b} or ${a\b} concatenates using filename separator, producing ${a}${/}${b} 
                     provided both ${a} and ${b} are non-empty

    ${a:b} or ${a;b} concatenates using path separator, producing ${a}${:}${b} 
                     provided both ${a} and ${b} are non-empty

#### Fileset expressions

If ${f} is a fileset, 

    ${f.dir}         base directory of the fileset
    ${f.filename}    full filename including ${f.dir}
    ${f.absolute}    absolute filename
    ${f.relative}    filename relative to ${f.dir}
    ${f.last}        trailing component of ${f.filename}
    ${f.path}        directory containing ${f.last}

For filesets defined with a pattern, the base directory is the portion of the path before the first wildcard. Building on earlier examples, and assuming the file src/com/example/hello.java exists,

    ${src.dir}       src
    ${src.filename}  src/com/example/hello.java
    ${src.relative}  com/example/hello.java
    ${src.last}      hello.java
    ${src.path}      src/com/example
    ${javac}         C:\windows\system32\java.exe
    ${javac.options} (multi-valued string)

#### Expression evaluation

Treatment of single- and multi-valued strings and fileset components varies depending on the enclosing XML element in the build file. The &lt;cmd&gt; element turns multi-valued expressions into separate arguments, and turns empty lists into the absence of arguments. The &lt;opt&gt; element contributes nothing to the enclosing command if any expression within it was empty, and the &lt;path&gt; element concatenates multi-valued expressions using ${:} as a separator. For example, the "javac" task defined above would run this command:

    C:\Program Files\Java\jdk1.8.0\bin\javac.exe -g -Xlint -d bin -sourcepath src src/com/example/hello.java

Unlike Make, Hobson will not submit this command to the shell or command line interpreter. This means filenames containing spaces do not need to be quoted, and quote characters do not need to be escaped. More subtly, parsing of the space-separated arguments within &lt;cmd&gt; occurs before expression evaluation, and Hobson determines the argument array it will pass to the invoked process.

#### Explicit and implicit loops

In the examples above, ${src} is a fileset defined by a pattern that matches only one file, src/com/example/hello.java. If there was more than one file in the fileset, then ${src.filename} would have multiple values, and as described above the &lt;cmd&gt; element pass those values as separate arguments. Since ${src.dir} is an attribute of the fileset itself, it remains a single-valued string.

However, if ${f} is a multi-fileset, then ${f.dir} is a list of strings. For example:


    <target name="bin1" pattern="bin1/**/*.class"/>
    <target name="bin2" pattern="bin2/**/*.class"/>
    <step depends="bin1,bin2" produces="jar">
        <run task="jar"/>
    </step>

Within the jar task, ${input} is a multi-fileset equivalent to ${bin1,bin2}. The jar command uses a for-each loop to create a -C argument for each one:

    <task name="jar">
        <cmd>${jar} cf ${output.filename} 
            <arg foreach="${input}">-C ${input.dir} ${input.relative}</arg>
        </cmd>
    </task>

The &lt;arg&gt; is explicitly repeated for each fileset in the input, and within the &lt;arg&gt; there is an implicit loop around the relative filename.

The Java tools support the use of argument files to avoid exceeding limits on command line arguments. The &lt;argfile&gt; element creates such a file and makes its name available as ${argfile}.

    <task name="jar">
        <argfile>
            <arg foreach="${input}">-C ${input.dir} ${input.relative}</arg>
        </argfile>
        <cmd>
            ${jar} cf ${output.filename} @${argfile}
        </cmd>
    </task>

A similar &lt;tmpfile&gt; element creates temporary files without the syntactic particulars of a Java tooling argument file.
