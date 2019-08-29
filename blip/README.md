BLIP
Summary: A nested template language.


Requirements
    Python 2.7, *UNIX*

Installation

   Place the 'blip' executable at /usr/local/bin and set it to be
   owned by root/root and executable (755).

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
   as template nesting.  To invoke a template file from within another
   template use the syntax {{:template_name:}} . Invocations may be nested,
   limited only by the Python software function call limits on the host system.

   Here is a simple example:

        poem2.tmpl:
             It was the best of times.

        poem.tmpl:
             {{:poem2:}}
             It was the worst of times.

        $ blip poem
        It was the best of times.
        It was the worst of times.

   NOTE: Template file 'poem2.tmpl' is invoked using the form {{:template_name:}}
   without the need for the '.tmpl' extension, which is assumed.


3. Template Variables

   Variables can be set once and interpolated later, or used inside subsequent nested
   template invocations.  Variables declared are available in the defining scope and nested
   template scopes.  However, see "Preservation", below.

   Example: Template variable interpolation later in the file:

        greeting.tmpl:
             {{first := Abraham}}
             {{last  := Lincoln}}
             Hello {{first}} {{last}}

        $ blip greeting
        Hello Abraham Lincoln

   Example: Template variable interpolation inside a nested template

        greeting.tmpl:
             Hello {{first}} {{last}}

        president.tmpl:
             {{first := Abraham}}
             {{last  := Lincoln}}
             {{:greeting:}}

        $ blip president
        Hello Abraham Lincoln


4. Template Parameters

   Nested templates may receive an arbitrary list of parameters, passed at invocation.
   They are treated, within the invoked template, the same as other variables.

   Each parameter's value may be a string or a list of strings (see below) but otherwise
   is untyped.  See Interpolation Scope, below, for more details on how parameters and other
   variables are handled.

   Parameters may be given in two forms: json key/values or inline assignment. In json
   format, the value given is text. In inline assigment, the value given is taken from
   a variable already defined in the calling scope. This is useful as a nmemonic device
   when a variable in a calling scope has a different name than the required on in a
   template being invoked.

       json key/values take the form { "varname" : "value", ... }
       inline assigment take the form !formalparm=some_variable ...}
   
   Example: json key/values

        greeting.tmpl:
             Hello {{first_name}} {{last_name}}

        b.tmpl:
             {{:greeting: { "first_name" : "Abraham", "last_name" : "Lincoln" } }}

        $ blip b
        Hello Abraham Lincoln

   Example: inline assignment

        greeting.tmpl:
            Hello {{first_name}} {{last_name}}

        b.tmpl:
            {{first := Abraham}}
            {{last  := Lincoln}}
            {{:greeting: !first_name=first !last_name=last}}

        $ blip b
        Hello Abraham Lincoln

        NOTE: If the name of the variable in the nested template is the same as in the calling template,
        the arguments can be omitted from the call ( e.g. {{:greeting:}} ). See 'Template Variables as Implicit
        Arguments', below.


5. Template Variables as Explicit Arguments

   As shown above, template variables may be passed explicitly.  In this case,
   the variables are available only within the called scope (and any nested scopes).

   Example: Variable value passed explicitly to nested template:

        greeting.tmpl:
             Hello {{first_name}} {{last_name}}

        b.tmpl:
             {{first := Abraham}}
             {{last  := Lincoln}}
             {{:greeting: { "first_name" : "{{first}}", "last_name" : "{{last}}" } }}

        $ blip b
        Hello Abraham Lincoln


6. Template Variables as Implicit Arguments

   Template variables may be passed implicitly.  This is simply taking advantage
   of the automatic nesting of scopes, where variables defined in a calling scope
   are available in nested scopes.

   Example: Template variable taken from the calling scope:
        greeting.tmpl:
             Hello {{first_name}} {{last_name}}

        b.tmpl:
             {{first_name := Abraham}}
             {{last_name  := Lincoln}}
             {{:greeting:}} {{# Calling scope contains the values needed.}}

        $ blip b
        Hello Abraham Lincoln


7. Interpolation Scope

   Each template interpolation has available to it all variable names defined in the current
   scope all the way to the top-level scope. The value of the variable is taken from the scope
   which set the variable. If set multiple times in the same scope, the most recent setting is
   used.  If a variable name is set as an argument to a nested template, it takes precedence
   over the calling scope, but the called scope can also override the value. An illustration
   of this is as follows:

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
             {{:outer:}}

        $ blip top
        "a" has value innerA2
        "b" has value outerB
        "c" has value argumentC
        "d" has value innerD
        "e" has value topE


8. List Parameters.

   If one (or more) of the explicit template arguments is a list, the nested template
   interpolation is performed once for each value in the list.  If more than one explicit
   nested template argument is a list, the corresponding value from each explicit nested
   template variable list will be used when interpolating the template once for each value. 
   All list arguments must have the same number of entries. Here is a contrived, albeit not
   very elegant, example:

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

   Implicitly passed Variables may contain lists, just like parameters. However, such variables will not
   result in multiple interpolations of called templates, as in the following example.  (Remember:
   variables from outer scopes are available in inner scopes):

        a.tmpl:
             This is my name: {{first_name}} {{last_name}}

        c.tmpl
             {{first_name := ["Abraham", "Benjamin"]}}
             {{last_name  := ["Lincoln", "Franklin"]}}
             {{:a:}} {{# Implicitly passed parameters 'first_name' and 'last_name'}}

        $ blip c
        This is my name: ["Abraham", "Benjamin"] ["Lincoln", "Franklin"]

   This happens because implicit parameters are not expanded as lists. Thus template 'a' is only
   interpolated once. Explicit parameters, as in the preceding example, would be required to extract
   list values one by one. See Also 'Variable Tagging Groups'.


10. Here-Templates

   Templates may defined inside a template file.  Once defined (with a leading <), they may be
   invoked as regular variables or, using a special syntax (the addition of a leading (<)) as
   a template.

   Example

        greeting.tmpl:
             {{<greet := This is my name: {{first_name}} {{last_name}} }}
             {{:<greet: { "first_name" : "Abraham",  "last_name" : "Lincoln"  } }}
             {{:<greet: { "first_name" : "Benjamin", "last_name" : "Franklin" } }}
             {{greet}} # treat as a variable, for example

        $ blip greeting
        This is my name: Abraham Lincoln
        This is my name: Benjamin Franklin 
        This is my name: {{first_name}} {{last_name}} # treat as a variable, for example

   NOTE: Here templates are scoped like variables, so they can be changed in inner scopes,
   or defined in outer scopes and invoked in inner ones. Note that Here templates may not
   be explicitly passed in nested template invocations, as variables can.


11. Preservation

   It is possible to preserve variable and here-template definitions made inside nested templates.
   This is useful, for example, when defining a set of variables or here-templates within a template
   file 'library' to be used by other templates. To do this, the nested template changes may be *preserved*
   in the calling scope. Use two colons to preserve when invoking the nested template, as shown
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

   It is possible to define a variable containing the name of another variable or template to expand. Indirection
   may, itself, be nested, as shown below.

   Here is an example of indirection:

   Example:

        embedded.tmpl:
             This is an embedded template.

        hosts.tmpl:
             {{host1 := lois}}
             {{host2 := clark}}
             {{host  := host1}}
             {{system := host}}
             {{tname := embedded}}
             {{fname := tname}}
             Nested Variable Indirection Output:
             {{!host}}
             {{!!system}} # ! indirection nesting supported

             Nested Indirect Template Invocation:
             {{:!!fname:}}

             Indirect Here-Template Invocation:
             {{<mytemp := This is really strange}}
             {{which := mytemp}}
             {{who := which}}
             {{:<!which:}} # Indirect invocation
             {{:<!!who:}} # Nested indirect invocation

        $ blip hosts
        Nested Variable Indirection Output:
        lois
        lois # ! indirection nesting supported

        Nested Indirect Template Invocation:
        This is an embedded template.

        Indirect Here-Template Invocation:
             
        This is really strange # Indirect invocation
        This is really strange # Nested indirect invocation

    NOTE: It is possible to achieve the same effect by (less-elegant) nesting, as follows:
          {{ {{ host }} }}
          {{ {{ {{ system }} }} }}

          Using this method, Template invocation indirection may also be nested:
          {{tname := embedded}}
          {{foo := tname}}
          {{: {{ {{ foo }} }} :}}  # This resolves to {{:embedded:}}


13. Variable Tagging Groups

   Variables may be grouped and used together using a tagging notation.  By tagging variables, the tag
   may be used in lieu of the variable names to refer to all the identically tagged variable names as
   a list.  To specify this, the tag appears after a second ':=', as in the example. Note that the resulting
   list is itself just a regular template variable, and is subject to the same rules and behaviors as other
   (list) variables.  Thus, for example, passing the tag variable explicitly to a template will result in
   repeated interpolations of the called template once for each tagged variable. See "Using Variable Tagging
   Groups with Nested Templates", below, for a special syntax which may be used with variable tagging groups.

   Example:

        hosts.tmpl:
             {{host1 := 10.0.0.3 := host}}
             {{host2 := 10.0.0.4 := host}}
             {{host3 := 10.0.0.5 := host}}
             {{host}}

        $ blip hosts
        ["host1", "host2", "host3"]


14. Using Variable Tagging Groups with Nested Templates

   A variable tagging group may be passed to a nested template, much as a list can.  Using a special
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

            {{# Note that the parameter name 'host' used inside 'who.tmpl' must match the group name used when
              invoking 'who' from 'main.tmpl'}}

        $ blip main
        ssh -l root 10.0.0.3 who # check host1
        ssh -l root 10.0.0.4 who # check host2
        ssh -l root 10.0.0.5 who # check host3


15. Template Argument Naming

   When the variable containing a needed value doesn't match the name expected in a nested template, explicitly
   naming the target variable solves the problem. See "main.tmpl" in the example below.

   Example:

        hosts.tmpl:
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


16. Quoting

    Quoting may be used to prevent blip from treating certain symbols (such as '{{' or ':=') specially.  For '{{' and '}}'
    use '{{{{' or '}}}}', respectively. To quote ':=' use ':=='.

17. Variable Group Generator Templates.

   A template may be defined which can generate a new variable group based on an existing one.  This is possible to do
   in a template file using features described above. First, use template variables to define a group.  Then expand
   the group by passing it explicitly to a template which in turn creates a new variable group containing entries for
   each of original variable group members. This is useful, for example, to expand a simple list of values to a more
   complicated data set. Since the resulting group and the original have the same size, they could be passed together
   to a template which combines the values to output something useful.

   Example:
        Generate a new variable group containing sshpass commands for each host in a previously-defined group.

        {{# Group Generator Template File}}
        sshpass-cmd.tmpl:
             {{sshpass-{{hostname}}-cmd := sshpass -p {{auth}} -t -l root {{!hostname}} := sshpass-group }}


        {{# Original group }}
        hosts.tmpl:
             {{host1 := 10.0.0.3 := lab22}}
             {{host2 := 10.0.0.4 := lab22}}
             {{host3 := 10.0.0.5 := lab22}}

        {{# Nested template invocation passing the original a tagging group }}
        main.tmpl:
             {{::sshpass-cmd:: { "auth" : "secret1" } !hostname=lab22}} {{# create variable group 'sshpass-group'}}

        The result is a new variable group but with more detailed structure. Note the use of 
        double-colons when invoking :sshpass-cmd: to preserve the results.


        Using quoting, sshpass-cmd may be defined as a here-template rather than in a file:
        {{# Group Generator Here-Template}}
         {{<sshpass-cmd := {{ ssh-{{hostname}}-cmd :== sshpass -p {{auth}} -t -l root {{!hostname}} :== sshpass-group }} }}

        ---

        We can now use the original group and the new group as parameters to a template to output something useful:

        Suppose, for example, we want to render text as input to a menuing tool that takes json output in the following format:

             { "name" : "NAMETEXT", "title" : "TITLETEXT", "cmd" : "CMDTEXT" }

        where "NAMETEXT" is displayed in a drop down menu, "TITLETEXT" appears in the window opened when the item is selected,
        and "CMDTEXT" is executed in a shell attached to the opened window.  The BLIP template to generate this would be:

             {{<menu-item := { "name": "{{name}}", "title": "{{name}}", "cmd" : "{{!cmd}}" }, }}

             NOTE: 'cmd' is given with an indirection symbol in '<menu-item'. That's because the value of 'cmd' will come from a
             tagging group containing the command list (e.g. 'sshpass-group')

        Finally, we invoke the '<menu-item' template with the host list and ssh-pass command list to combine into menu items:

             {{:<menu-item: !name=lab22 !cmd=sshpass-group}}

        Since both groups have the same length (required when passing lists or groups), this results in the following output:


        $ blip main.tmpl
        { "name": "host1", "title": "host1", "cmd" : "sshpass -p secret1 -t -l root 10.0.0.3" },
        { "name": "host2", "title": "host2", "cmd" : "sshpass -p secret1 -t -l root 10.0.0.4" },
        { "name": "host3", "title": "host3", "cmd" : "sshpass -p secret1 -t -l root 10.0.0.5" },


MORE: JSON assignments, Dictionary -> Tagged Assignment Conversion.
