BLIP
Summary: A nested template language


Installation

   Place the 'blip' executable at /usr/local/bin and set it to be
   executable (755).  As a system-wide utility it is probably
   appropriate that the installed file '/usr/local/bin/blip' be owned by root.


Features

1. Template Files

   Template files end with the extension .tmpl, which is assumed
   when using BLIP.  So if there is a template file named
   'declare.tmpl', it can be invoked as 'declare'.

   Example: A template file containing simple text.

        declare.tmpl:
             We hold these truths to be self-evident...

        $ blip declare
        We hold these truths to be self-evident...

   Template files are searched using the BLIP_PATH environment
   variable, if set. Otherwise, the current directory is searched.


2. Template Nesting

   A template file may invoke other template files, causing the invoked
   template to be interpolated at the point of invocation. This is known
   as template nesting.  Nesting depth is limited only by the
   Python software function call limits on the host system.

   Here is a simple example:

        poem2.tmpl:
             It was the best of times.

        poem.tmpl:
             {{:poem2:}}
             It was the worst of times.

        $ blip poem
        It was the best of times.
        It was the worst of times.

   NOTE: Template file 'poem2.tmpl' is invoked using the form {{:<template_name>:}}
   without the need for the '.tmpl' extension, which is assumed.


3. Template Variables

   Variables can be set once and interpolated later in the file, or used inside nested templates.

   Example: Template variable interpolation later in the file
        greeting.tmpl:
             {{first := Abraham}}
             {{last  := Lincoln}}
             Hello, {{first}} {{last}}

        $ blip greeting
        Hello Abraham Lincoln

   Variables defined before invoking a nested template will be visible in the nested template.


4. Template Parameters

   Nested templates may receive parameters, passed at invocation.
   Parameter values are strings (or lists, see below) but otherwise
   are untyped.  If a required parameter is not passed, or previously
   defined, an error will result. See Interpolation Scope for details

   Parameter lists consist of json key/values or variable names:

   Example: json key/values

        greeting.tmpl:
             Hello {{first_name}} {{last_name}}

        b.tmpl:
             {{:greeting: { "first_name" : "Abraham", "last_name" : "Lincoln" } }}

        $ blip b
        Hello Abraham Lincoln

   Example: variable names

        greeting.tmpl:
            Hello {{first_name}} {{last_name}}

        b.tmpl:
            {{first := Abraham}}
            {{last  := Lincoln}}
            {{:greeting: !first_name=first !last_name=last}}

        $ blip b
        Hello Abraham Lincoln

        NOTE: If the name of the variable in the nested template is the same as in the calling template,
        the arguments can be omitted from the call ( e.g. {{:greeting:}} ). This is due to the scoping rules
        for BLIP, which are explained below.


5. Template Variables as Explict Arguments

   Example: Template variable value passed explicitly to nested template:

        greeting.tmpl:
             Hello {{first_name}} {{last_name}}

        b.tmpl:
             {{first := Abraham}}
             {{last  := Lincoln}}
             {{:greeting: { "first_name" : {{first}}, "last_name" : {{last}} } }}

        $ blip b
        Hello Abraham Lincoln


6. Template Variables as Implicit Arguments

   Example: Template variable taken from outer scope:
        greeting.tmpl:
             Hello {{first_name}} {{last_name}}

        b.tmpl:
             {{first_name := Abraham}}
             {{last_name  := Lincoln}}
             {{:greeting:}} {{# Outer scope contains the values needed..}}

        $ blip b
        Hello Abraham Lincoln


7. Interpolation Scope

   Each template interpolation has available to it all variable names defined in the current
   scope all the way to the top-level scope. The value of the variable is taken from
   the innermost scope which set the variable. If set multiple times in the same
   scope, the most recent setting is used.  If a variable name is set as an argument to
   a nested template, it takes precedence over the calling scope, but the called scope can over-
   ride the value. An example of this is as follows:

   Example:
        inner.tmpl:
             {{a := innerA}}
             {{d := innerD}}
             {{a := innerA2}}
             "a" has value {{a}}
             "b" has value {{b}}
             "c" has value {{c}}
             "d" has value {{d}}
             "e" has value {{e}}

        outer.tmpl:
             {{a := outerA}}
             {{b := outerB}}
             {{c := outerC}}
             {{d := outerD}}
             {{:inner: { "c" : "argumentC", "d" : "argumentD" } }}

        top.tmpl:
             {{a := topA}}
             {{b := topB}}
             {{c := topC}}
             {{d := topD}}
             {{e := topE}}

        $ blip top
        "a" has value innerA2
        "b" has value outerB
        "c" has value argumentC
        "d" has value innerD
        "e" has value topE


8. List Parameters.

   If one of the explicit template arguments is a list, the nested template interpolation
   is performed once for each value in the list.  If more than one explicit nested
   template argument is a list, the corresponding value from each explicit nested template
   variable list will be used when interpolating the template once for each value. At
   present, all list arguments must have the same number of entries.

        name.tmpl:
             This is my name: {{title}} {{first_name}} {{last_name}}

        b.tmpl:
             {{:name: { "title" : "Mr.", "first_name" : ["Abraham", "Benjamin"], "last_name" : ["Lincoln", "Franklin"]} }}

        $ blip b
        This is my name: Mr. Abraham Lincoln
        This is my name: Mr. Benjamin Franklin

        NOTE: In this example there must be a blank line at the end of 'name.tmpl'. This is required for the
        two interpolations to be output on separate lines, otherwise the resulting multiple interpolations
        would output: 'This is my name: Abraham LincolnThis is my name: Benjamin Franklin'.


9. List Variables

   Variables may contain lists, just like parameters. However, such variables will not result in multiple
   interpolations of called templates unless they are passed explicitly, as in the following example.

        a.tmpl:
             This is my name: {{first_name}} {{last_name}}

        b.tmpl:
             {{firsts := ["Abraham", "Benjamin"]}}
             {{lasts  := ["Lincoln", "Franklin"]}}
             {{:a: { "first_name : {{firsts}}, "last_name" : {{lasts}} } }} {{# Explicitly-passed}}

        $ blip b

   Here is what happens when attempting to use the variables directly.
   (Remember: variables from outer scopes are available in inner scopes):

        a.tmpl:
             This is my name: {{first_name}} {{last_name}}

        c.tmpl
             {{first_name := ["Abraham", "Benjamin"]}}
             {{last_name  := ["Lincoln", "Franklin"]}}
             {{:a:}} {{# Implicitly passed}}

        $ blip c
        This is my name: ["Abraham", "Benjamin"] ["Lincoln", "Franklin"]

   This happens because template 'a' is only interpolated once. Explicit parameters, as in the preceding
   example, are required to extract list values one by one.


10. Here-Templates

   Templates may defined inside a template file.  Once defined, they may be invoked like
   template files with a slightly different syntax (the addition of a leading (<)).

   Example

        greeting.tmpl:
             {{<greet := This is my name: {{first_name}} {{last_name}} }}
             {{:<greet: { "first_name" : "Abraham",  "last_name" : "Lincoln"  } }}
             {{:<greet: { "first_name" : "Benjamin", "last_name" : "Franklin" } }}

        $ blip greeting
        This is my name: Abraham Lincoln
        This is my name: Benjamin Franklin

   NOTE: Here templates are scoped like variables, so they can be changed in inner scopes,
   or defined in outer scopes and invoked in inner ones. Here templates may not be explicitly
   defined in nested template invocations, as variables can.


11. Preservation

   It is possible to preserve variable and here-template definitions made in nested templates.
   This is useful, for example, when defining a set of variables or here-templates in a template
   file to be used by other templates. To do this, the nested template changes must be *preserved*
   in the calling scope. To preserve, use two colons when invoking the nested template, as shown
   in the following example.

   Example:

        library.tmpl:
             {{<greet := "This is my name {{first_name}} {{last_name}} }}

        welcome.tmpl:
             {{::library::}}  {{# Preserve the definition of '<greet'}}
             {{:<greet: { "first_name" : "Abraham",  "last_name" : "Lincoln"  } }}
             {{:<greet: { "first_name" : "Benjamin", "last_name" : "Franklin" } }}

        $ blip welcome
        This is my name: Abraham Lincoln
        This is my name: Benjamin Franklin


12. Variable Indirection

   It is possible to define a variable containing the name of another variable whose value is desired.
   Here is an example of indirection:

   Example:

        hosts.tmpl:
             {{host1 := lois}}
             {{host2 := clark}}
             {{host  := host1}}

             {{!host}}

        $ blip hosts
        lois

    NOTE: It is possible to achieve the same effect by nesting, as follows:
          {{ {{ host }} }}


13. Variable Tagging Groups

   Variables may be grouped and used together using a tagging notation.  By tagging variables, the tag
   may be used in lieu of the variable names to refer to all the identically tagged variable names as
   a list.  To specify this, the tag appears after a second ':=', as in the example.

   Example:

        hosts.tmpl:
             {{host1 := 10.0.0.3 := host}}
             {{host2 := 10.0.0.4 := host}}
             {{host3 := 10.0.0.5 := host}}
             {{host}}

        $ blip hosts
        ["host1", "host2", "host3"]


14. Using Variable Groups with Nested Templates

   A variable group may be passed to a nested template, much as a list can.  Using a special
   syntax, the nested template will be interpolated multiple times. With each interpolation,
   the value of the resulting parameter will be the name of each variable in the group.  This
   variable, in turn, may be accessed using indirection.  To simplify understanding of this concept,
   see the following example.

   Example:

        hosts.tmpl:
             {{# Create a variable group called 'host'}}
             {{host1 := 10.0.0.3 := host}}
             {{host2 := 10.0.0.4 := host}}
             {{host3 := 10.0.0.5 := host}}

        who.tmpl:
            {{# a simple command to run against each host}}
            ssh -l root {{!host}} who # check {{host}}

        main.tmpl:
             {{::hosts::}}      {{# define the 'host' variable group. Use double-colon to preserve the group}}
             {{:who: !host}}    {{# pass each variable name in the group 'host' to 'who.tmpl' parameter of the same name}}

            {{# Note that the parameter name 'host' used in 'who.tmpl' must match the group name used in 'main.tmpl'}}

        $ blip main
        ssh -l root 10.0.0.3 who # check host1
        ssh -l root 10.0.0.4 who # check host2
        ssh -l root 10.0.0.5 who # check host3


15. Template Argument Naming

   When the variable containing a needed value doesn't match the name expected in a nested template, explicitly
   naming the target variable solves the problem. See "main.tmpl" in the example below.

   Example:

        fishtanks.tmpl:
             {{# Create a variable group called 'tank'}}
             {{tank1 := 10.0.0.3 := tank}}
             {{tank2 := 10.0.0.4 := tank}}
             {{tank3 := 10.0.0.5 := tank}}

        who.tmpl:
             ssh -l root {{!host}} who # check {{host}}

        main.tmpl:
             {{::hosts::}}           {{# define the 'host' variable group. Use double-colon to preserve the group}}
             {{:who: !host=tank}}    {{# pass each variable name in the group 'tank' to 'who.tmpl' as 'host'}}

        $blip main
        ssh -l root 10.0.0.3 who # check tank1
        ssh -l root 10.0.0.4 who # check tank2
        ssh -l root 10.0.0.5 who # check tank3


16. Here-Template Variable Group Generators.

   A template may be defined which can generate a new variable group based on an existing one.  This is easy to do
   in a template file using features described above. Use template variables to define the resulting variable name,
   variable value, and the variable group.  This is useful, for example, to expand a simple list of values to a more
   complicated data set. Since the resulting group and the original have the same size, they could be passed together
   to a template which combines the values to output something useful.

   Example:
        Generate a variable group containing sshpass commands for each host in a list.

        sshpass-cmd.tmpl:
             {{sshpass-{{hostname}}-cmd := sshpass -p {{auth}} -t -l root {{!hostname}} := sshpass-group }}


        hosts.tmpl:
             {{host1 := 10.0.0.3 := lab22}}
             {{host2 := 10.0.0.4 := lab22}}
             {{host3 := 10.0.0.5 := lab22}}

        main.tmpl:
             {{::sshpass-cmd:: { "auth" : "secret1" } !hostname=lab22}} {{# create variable group 'sshpass-group'}}

        The result is a new variable group with the same number of entries as the original, but with more
        detailed structure.


        The same thing can be accomplished using a here-template for sshpass-cmd. In this case though, to distinguish
        between the here-template assignment and the group variable assignment, both of which are ':=', an additional
        '=' is added for the internal assignment symbols.  You should be able to spot the two ':==' within the text here:

             {{<sshpass-cmd := {{ ssh-{{hostname}}-cmd :== sshpass -p {{auth}} -t -l root {{!hostname}} :== sshpass-group }} }}

        We can now use the original list and new list in a combined call to a final template which uses the lists to output
        something.

        Suppose, for example, we want to render text as input to a menuing tool that takes json output:

             { "name" : "NAMETEXT", "title" : "TITLETEXT", "cmd" : "CMDTEXT" }

        where "NAMETEXT" is displayed in a drop down menu, "TITLETEXT" appears in the window opened when the item is selected,
        and "CMDTEXT" is executed in a shell attached to the opened window.  The BLIP code reflecting this would be:

             {{<menu-item := { "name": "{{name}}", "title": "{{name}}", "cmd" : "{{!cmd}}" }, }}

             NOTE: 'cmd' is given with an indirection symbol in '<menu-item'. That's because the value of 'cmd' will come from the
             'sshpass-group' list of variables containing sshpass commands.

        Finally, we invoke our group-generator with a list of hostnames to create variable group 'sshpass-group',
        and invoke the '<menu-item' template with the host list and ssh-pass command list to combine into menu items:

             {{::<sshpass-cmd:: { "auth" : "secret1" } !hostname=lab22}}
             {{:<menu-item: !name=lab22 !cmd=sshpass-group}}


        $ blip main.tmpl
        { "name": "host1", "title": "host1", "cmd" : "sshpass -p secret1 -t -l root 10.0.0.3" },
        { "name": "host2", "title": "host2", "cmd" : "sshpass -p secret1 -t -l root 10.0.0.4" },
        { "name": "host3", "title": "host3", "cmd" : "sshpass -p secret1 -t -l root 10.0.0.5" },
