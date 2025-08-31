---
title: "فایل‌های تنظیمات مناسب ایجاد پروژه جدید در .Net"
categories:
  - Net
tags:
  - net
  - good_file
  - startup
---

برای اینکه پروژه شما نظرم بیشتری داشته باشد می‌توانید فایل‌های زیر را طبق ساختار گفته شده به پروژه خود اضافه کنید. هر کدام از فایل‌ها تنظیمات خاصی را انجام می‌هد که در ادامه گفته شده است.  

`files`

```
-> root (where .git exist)
    -> .editorconfig
    -> .gitattributes
    -> .gitignore
    -> BannedSymbols.txt
    -> nuget.config
    -> pull_request_template.md
        
        -> src/Prj (where sln exist)
            -> Directory.Build.props
            -> Directory.Build.targets
            -> Directory.Packages.props
            -> runtimeconfig.template.json
```

برای یکسان سازی تنظیمات IDE مانند طول هر Tab یا موارد ظاهری دیگر
.editorconfig`
```
root = true

[*.MyGenerated.cs]
generated_code = true

# All files
[*]
indent_style = space

# Xml files
[*.{xml,csproj,props,targets,ruleset,nuspec,resx}]
indent_size = 2

# Javascript files
[*.js]
indent_size = 2

# Json files
[*.{json,config,nswag}]
indent_size = 2

# C# files
[*.cs]

#### Core EditorConfig Options ####

# Indentation and spacing
indent_size = 4
tab_width = 4

# New line preferences
end_of_line = crlf
insert_final_newline = true

# Maximum line length for code formatting
max_line_length = 180

#### .NET Coding Conventions ####
[*.{cs,vb}]

# Organize usings
dotnet_separate_import_directive_groups = false
dotnet_sort_system_directives_first = true
file_header_template = unset

# this. and Me. preferences
dotnet_style_qualification_for_event = false:silent
dotnet_style_qualification_for_field = false:silent
dotnet_style_qualification_for_method = false:silent
dotnet_style_qualification_for_property = false:silent

# Language keywords vs BCL types preferences
dotnet_style_predefined_type_for_locals_parameters_members = true:silent
dotnet_style_predefined_type_for_member_access = true:silent

# Parentheses preferences
dotnet_style_parentheses_in_arithmetic_binary_operators = always_for_clarity:silent
dotnet_style_parentheses_in_other_binary_operators = always_for_clarity:silent
dotnet_style_parentheses_in_other_operators = never_if_unnecessary:silent
dotnet_style_parentheses_in_relational_binary_operators = always_for_clarity:silent

# Modifier preferences
dotnet_style_require_accessibility_modifiers = for_non_interface_members:silent

# Expression-level preferences
dotnet_style_coalesce_expression = true:suggestion
dotnet_style_collection_initializer = true:suggestion
dotnet_style_explicit_tuple_names = true:suggestion
dotnet_style_null_propagation = true:suggestion
dotnet_style_object_initializer = true:suggestion
dotnet_style_operator_placement_when_wrapping = beginning_of_line
dotnet_style_prefer_auto_properties = true:suggestion
dotnet_style_prefer_compound_assignment = true:suggestion
dotnet_style_prefer_conditional_expression_over_assignment = true:suggestion
dotnet_style_prefer_conditional_expression_over_return = true:suggestion
dotnet_style_prefer_inferred_anonymous_type_member_names = true:suggestion
dotnet_style_prefer_inferred_tuple_names = true:suggestion
dotnet_style_prefer_is_null_check_over_reference_equality_method = true:suggestion
dotnet_style_prefer_simplified_boolean_expressions = true:suggestion
dotnet_style_prefer_simplified_interpolation = true:suggestion

# Field preferences
dotnet_style_readonly_field = true:warning

# Parameter preferences
dotnet_code_quality_unused_parameters = all:suggestion

# Suppression preferences
dotnet_remove_unnecessary_suppression_exclusions = none

#### C# Coding Conventions ####
[*.cs]

# var preferences
csharp_style_var_elsewhere = false:silent
csharp_style_var_for_built_in_types = false:silent
csharp_style_var_when_type_is_apparent = false:silent

# Expression-bodied members
csharp_style_expression_bodied_accessors = true:silent
csharp_style_expression_bodied_constructors = false:silent
csharp_style_expression_bodied_indexers = true:silent
csharp_style_expression_bodied_lambdas = true:suggestion
csharp_style_expression_bodied_local_functions = false:silent
csharp_style_expression_bodied_methods = false:silent
csharp_style_expression_bodied_operators = false:silent
csharp_style_expression_bodied_properties = true:silent

# Pattern matching preferences
csharp_style_pattern_matching_over_as_with_null_check = true:suggestion
csharp_style_pattern_matching_over_is_with_cast_check = true:suggestion
csharp_style_prefer_not_pattern = true:suggestion
csharp_style_prefer_pattern_matching = true:silent
csharp_style_prefer_switch_expression = true:suggestion

# Null-checking preferences
csharp_style_conditional_delegate_call = true:suggestion

# Modifier preferences
csharp_prefer_static_local_function = true:warning
csharp_preferred_modifier_order = public,private,protected,internal,static,extern,new,virtual,abstract,sealed,override,readonly,unsafe,volatile,async:silent

# Code-block preferences
csharp_prefer_braces = true:silent
csharp_prefer_simple_using_statement = true:suggestion

# Expression-level preferences
csharp_prefer_simple_default_expression = true:suggestion
csharp_style_deconstructed_variable_declaration = true:suggestion
csharp_style_inlined_variable_declaration = true:suggestion
csharp_style_pattern_local_over_anonymous_function = true:suggestion
csharp_style_prefer_index_operator = true:suggestion
csharp_style_prefer_range_operator = true:suggestion
csharp_style_throw_expression = true:suggestion
csharp_style_unused_value_assignment_preference = discard_variable:suggestion
csharp_style_unused_value_expression_statement_preference = discard_variable:silent

# 'using' directive preferences
csharp_using_directive_placement = outside_namespace:silent

#### C# Formatting Rules ####

# New line preferences
csharp_new_line_before_catch = true
csharp_new_line_before_else = true
csharp_new_line_before_finally = true
csharp_new_line_before_members_in_anonymous_types = true
csharp_new_line_before_members_in_object_initializers = true
csharp_new_line_before_open_brace = all
csharp_new_line_between_query_expression_clauses = true

# Indentation preferences
csharp_indent_block_contents = true
csharp_indent_braces = false
csharp_indent_case_contents = true
csharp_indent_case_contents_when_block = true
csharp_indent_labels = one_less_than_current
csharp_indent_switch_labels = true

# Space preferences
csharp_space_after_cast = false
csharp_space_after_colon_in_inheritance_clause = true
csharp_space_after_comma = true
csharp_space_after_dot = false
csharp_space_after_keywords_in_control_flow_statements = true
csharp_space_after_semicolon_in_for_statement = true
csharp_space_around_binary_operators = before_and_after
csharp_space_around_declaration_statements = false
csharp_space_before_colon_in_inheritance_clause = true
csharp_space_before_comma = false
csharp_space_before_dot = false
csharp_space_before_open_square_brackets = false
csharp_space_before_semicolon_in_for_statement = false
csharp_space_between_empty_square_brackets = false
csharp_space_between_method_call_empty_parameter_list_parentheses = false
csharp_space_between_method_call_name_and_opening_parenthesis = false
csharp_space_between_method_call_parameter_list_parentheses = false
csharp_space_between_method_declaration_empty_parameter_list_parentheses = false
csharp_space_between_method_declaration_name_and_open_parenthesis = false
csharp_space_between_method_declaration_parameter_list_parentheses = false
csharp_space_between_parentheses = false
csharp_space_between_square_brackets = false

# Wrapping preferences
csharp_preserve_single_line_blocks = true
csharp_preserve_single_line_statements = true
csharp_style_namespace_declarations = file_scoped:silent
csharp_style_prefer_method_group_conversion = true:silent
csharp_style_prefer_top_level_statements = true:silent
csharp_style_prefer_primary_constructors = true:suggestion
csharp_style_prefer_null_check_over_type_check = true:suggestion
csharp_style_prefer_local_over_anonymous_function = true:suggestion
csharp_style_implicit_object_creation_when_type_is_apparent = true:suggestion
csharp_style_prefer_tuple_swap = true:suggestion
csharp_style_prefer_utf8_string_literals = true:suggestion

#### Naming styles ####
[*.{cs,vb}]

# Naming rules

dotnet_naming_rule.types_and_namespaces_should_be_pascalcase.severity = suggestion
dotnet_naming_rule.types_and_namespaces_should_be_pascalcase.symbols = types_and_namespaces
dotnet_naming_rule.types_and_namespaces_should_be_pascalcase.style = pascalcase

dotnet_naming_rule.interfaces_should_be_ipascalcase.severity = suggestion
dotnet_naming_rule.interfaces_should_be_ipascalcase.symbols = interfaces
dotnet_naming_rule.interfaces_should_be_ipascalcase.style = ipascalcase

dotnet_naming_rule.type_parameters_should_be_tpascalcase.severity = suggestion
dotnet_naming_rule.type_parameters_should_be_tpascalcase.symbols = type_parameters
dotnet_naming_rule.type_parameters_should_be_tpascalcase.style = tpascalcase

dotnet_naming_rule.methods_should_be_pascalcase.severity = suggestion
dotnet_naming_rule.methods_should_be_pascalcase.symbols = methods
dotnet_naming_rule.methods_should_be_pascalcase.style = pascalcase

dotnet_naming_rule.properties_should_be_pascalcase.severity = suggestion
dotnet_naming_rule.properties_should_be_pascalcase.symbols = properties
dotnet_naming_rule.properties_should_be_pascalcase.style = pascalcase

dotnet_naming_rule.events_should_be_pascalcase.severity = suggestion
dotnet_naming_rule.events_should_be_pascalcase.symbols = events
dotnet_naming_rule.events_should_be_pascalcase.style = pascalcase

dotnet_naming_rule.local_variables_should_be_camelcase.severity = suggestion
dotnet_naming_rule.local_variables_should_be_camelcase.symbols = local_variables
dotnet_naming_rule.local_variables_should_be_camelcase.style = camelcase

dotnet_naming_rule.local_constants_should_be_camelcase.severity = suggestion
dotnet_naming_rule.local_constants_should_be_camelcase.symbols = local_constants
dotnet_naming_rule.local_constants_should_be_camelcase.style = camelcase

dotnet_naming_rule.parameters_should_be_camelcase.severity = suggestion
dotnet_naming_rule.parameters_should_be_camelcase.symbols = parameters
dotnet_naming_rule.parameters_should_be_camelcase.style = camelcase

dotnet_naming_rule.public_fields_should_be_pascalcase.severity = suggestion
dotnet_naming_rule.public_fields_should_be_pascalcase.symbols = public_fields
dotnet_naming_rule.public_fields_should_be_pascalcase.style = pascalcase

dotnet_naming_rule.private_fields_should_be__camelcase.severity = suggestion
dotnet_naming_rule.private_fields_should_be__camelcase.symbols = private_fields
dotnet_naming_rule.private_fields_should_be__camelcase.style = _camelcase

dotnet_naming_rule.private_static_fields_should_be_s_camelcase.severity = suggestion
dotnet_naming_rule.private_static_fields_should_be_s_camelcase.symbols = private_static_fields
dotnet_naming_rule.private_static_fields_should_be_s_camelcase.style = s_camelcase

dotnet_naming_rule.public_constant_fields_should_be_pascalcase.severity = suggestion
dotnet_naming_rule.public_constant_fields_should_be_pascalcase.symbols = public_constant_fields
dotnet_naming_rule.public_constant_fields_should_be_pascalcase.style = pascalcase

dotnet_naming_rule.private_constant_fields_should_be_pascalcase.severity = suggestion
dotnet_naming_rule.private_constant_fields_should_be_pascalcase.symbols = private_constant_fields
dotnet_naming_rule.private_constant_fields_should_be_pascalcase.style = pascalcase

dotnet_naming_rule.public_static_readonly_fields_should_be_pascalcase.severity = suggestion
dotnet_naming_rule.public_static_readonly_fields_should_be_pascalcase.symbols = public_static_readonly_fields
dotnet_naming_rule.public_static_readonly_fields_should_be_pascalcase.style = pascalcase

dotnet_naming_rule.private_static_readonly_fields_should_be_pascalcase.severity = suggestion
dotnet_naming_rule.private_static_readonly_fields_should_be_pascalcase.symbols = private_static_readonly_fields
dotnet_naming_rule.private_static_readonly_fields_should_be_pascalcase.style = pascalcase

dotnet_naming_rule.enums_should_be_pascalcase.severity = suggestion
dotnet_naming_rule.enums_should_be_pascalcase.symbols = enums
dotnet_naming_rule.enums_should_be_pascalcase.style = pascalcase

dotnet_naming_rule.local_functions_should_be_pascalcase.severity = suggestion
dotnet_naming_rule.local_functions_should_be_pascalcase.symbols = local_functions
dotnet_naming_rule.local_functions_should_be_pascalcase.style = pascalcase

dotnet_naming_rule.non_field_members_should_be_pascalcase.severity = suggestion
dotnet_naming_rule.non_field_members_should_be_pascalcase.symbols = non_field_members
dotnet_naming_rule.non_field_members_should_be_pascalcase.style = pascalcase

# Symbol specifications

dotnet_naming_symbols.interfaces.applicable_kinds = interface
dotnet_naming_symbols.interfaces.applicable_accessibilities = public, internal, private, protected, protected_internal, private_protected
dotnet_naming_symbols.interfaces.required_modifiers = 

dotnet_naming_symbols.enums.applicable_kinds = enum
dotnet_naming_symbols.enums.applicable_accessibilities = public, internal, private, protected, protected_internal, private_protected
dotnet_naming_symbols.enums.required_modifiers = 

dotnet_naming_symbols.events.applicable_kinds = event
dotnet_naming_symbols.events.applicable_accessibilities = public, internal, private, protected, protected_internal, private_protected
dotnet_naming_symbols.events.required_modifiers = 

dotnet_naming_symbols.methods.applicable_kinds = method
dotnet_naming_symbols.methods.applicable_accessibilities = public, internal, private, protected, protected_internal, private_protected
dotnet_naming_symbols.methods.required_modifiers = 

dotnet_naming_symbols.properties.applicable_kinds = property
dotnet_naming_symbols.properties.applicable_accessibilities = public, internal, private, protected, protected_internal, private_protected
dotnet_naming_symbols.properties.required_modifiers = 

dotnet_naming_symbols.public_fields.applicable_kinds = field
dotnet_naming_symbols.public_fields.applicable_accessibilities = public, internal
dotnet_naming_symbols.public_fields.required_modifiers = 

dotnet_naming_symbols.private_fields.applicable_kinds = field
dotnet_naming_symbols.private_fields.applicable_accessibilities = private, protected, protected_internal, private_protected
dotnet_naming_symbols.private_fields.required_modifiers = 

dotnet_naming_symbols.private_static_fields.applicable_kinds = field
dotnet_naming_symbols.private_static_fields.applicable_accessibilities = private, protected, protected_internal, private_protected
dotnet_naming_symbols.private_static_fields.required_modifiers = static

dotnet_naming_symbols.types_and_namespaces.applicable_kinds = namespace, class, struct, interface, enum
dotnet_naming_symbols.types_and_namespaces.applicable_accessibilities = public, internal, private, protected, protected_internal, private_protected
dotnet_naming_symbols.types_and_namespaces.required_modifiers = 

dotnet_naming_symbols.non_field_members.applicable_kinds = property, event, method
dotnet_naming_symbols.non_field_members.applicable_accessibilities = public, internal, private, protected, protected_internal, private_protected
dotnet_naming_symbols.non_field_members.required_modifiers = 

dotnet_naming_symbols.type_parameters.applicable_kinds = namespace
dotnet_naming_symbols.type_parameters.applicable_accessibilities = *
dotnet_naming_symbols.type_parameters.required_modifiers = 

dotnet_naming_symbols.private_constant_fields.applicable_kinds = field
dotnet_naming_symbols.private_constant_fields.applicable_accessibilities = private, protected, protected_internal, private_protected
dotnet_naming_symbols.private_constant_fields.required_modifiers = const

dotnet_naming_symbols.local_variables.applicable_kinds = local
dotnet_naming_symbols.local_variables.applicable_accessibilities = local
dotnet_naming_symbols.local_variables.required_modifiers = 

dotnet_naming_symbols.local_constants.applicable_kinds = local
dotnet_naming_symbols.local_constants.applicable_accessibilities = local
dotnet_naming_symbols.local_constants.required_modifiers = const

dotnet_naming_symbols.parameters.applicable_kinds = parameter
dotnet_naming_symbols.parameters.applicable_accessibilities = *
dotnet_naming_symbols.parameters.required_modifiers = 

dotnet_naming_symbols.public_constant_fields.applicable_kinds = field
dotnet_naming_symbols.public_constant_fields.applicable_accessibilities = public, internal
dotnet_naming_symbols.public_constant_fields.required_modifiers = const

dotnet_naming_symbols.public_static_readonly_fields.applicable_kinds = field
dotnet_naming_symbols.public_static_readonly_fields.applicable_accessibilities = public, internal
dotnet_naming_symbols.public_static_readonly_fields.required_modifiers = readonly, static

dotnet_naming_symbols.private_static_readonly_fields.applicable_kinds = field
dotnet_naming_symbols.private_static_readonly_fields.applicable_accessibilities = private, protected, protected_internal, private_protected
dotnet_naming_symbols.private_static_readonly_fields.required_modifiers = readonly, static

dotnet_naming_symbols.local_functions.applicable_kinds = local_function
dotnet_naming_symbols.local_functions.applicable_accessibilities = *
dotnet_naming_symbols.local_functions.required_modifiers = 

# Naming styles

dotnet_naming_style.pascalcase.required_prefix = 
dotnet_naming_style.pascalcase.required_suffix = 
dotnet_naming_style.pascalcase.word_separator = 
dotnet_naming_style.pascalcase.capitalization = pascal_case

dotnet_naming_style.ipascalcase.required_prefix = I
dotnet_naming_style.ipascalcase.required_suffix = 
dotnet_naming_style.ipascalcase.word_separator = 
dotnet_naming_style.ipascalcase.capitalization = pascal_case

dotnet_naming_style.tpascalcase.required_prefix = T
dotnet_naming_style.tpascalcase.required_suffix = 
dotnet_naming_style.tpascalcase.word_separator = 
dotnet_naming_style.tpascalcase.capitalization = pascal_case

dotnet_naming_style._camelcase.required_prefix = _
dotnet_naming_style._camelcase.required_suffix = 
dotnet_naming_style._camelcase.word_separator = 
dotnet_naming_style._camelcase.capitalization = camel_case

dotnet_naming_style.camelcase.required_prefix = 
dotnet_naming_style.camelcase.required_suffix = 
dotnet_naming_style.camelcase.word_separator = 
dotnet_naming_style.camelcase.capitalization = camel_case

dotnet_naming_style.s_camelcase.required_prefix = s_
dotnet_naming_style.s_camelcase.required_suffix = 
dotnet_naming_style.s_camelcase.word_separator = 
dotnet_naming_style.s_camelcase.capitalization = camel_case

dotnet_style_namespace_match_folder = true:suggestion

dotnet_diagnostic.CA2007.severity = none
dotnet_diagnostic.CA1716.severity = none
dotnet_diagnostic.CA5351.severity = none
dotnet_diagnostic.CA1000.severity = none
dotnet_diagnostic.CA1848.severity = none
dotnet_diagnostic.CA1852.severity = none
dotnet_diagnostic.CA1859.severity = none
dotnet_diagnostic.CA1861.severity = none
dotnet_diagnostic.CA1001.severity = none
dotnet_diagnostic.CA1822.severity = none
dotnet_diagnostic.CA2201.severity = none
dotnet_diagnostic.NU1109.severity = none
dotnet_diagnostic.CS8618.severity = none

```

مواردی که در زمان کامیت جز تغییرات نمی‌آیند
`.gitignore`
```
*.swp
*.*~
project.lock.json
.DS_Store
*.pyc
nupkg/

# Visual Studio Code
.vscode

# Rider
.idea

# User-specific files
*.suo
*.user
*.userosscache
*.sln.docstates

# Build results
[Dd]ebug/
[Dd]ebugPublic/
[Rr]elease/
[Rr]eleases/
x64/
x86/
build/
bld/
[Bb]in/
[Oo]bj/
[Oo]ut/
msbuild.log
msbuild.err
msbuild.wrn

# Visual Studio 2015
.vs/

appsettings.Development.json
```

لیست مواردی که در پروژه امکان استفاده از آنها وجود ندارد.  
`BannedSymbols.txt`
```
# Disallow usage of DateTime.Now
T:System.DateTime
M:System.DateTime.Now -> Do not use DateTime.Now. Use IClock.Now instead.
M:System.DateTime.UtcNow -> Use IClock.UtcNow or another abstraction.
```

یک csproj بیس برای تمام پروژه‌های موجود که می‌توانید مواردی مانند نال‌‌پذیر بودن یا ورژن .net را در آن مشخص کنید.  
`Directory.Build.props`
```
<Project>
  <PropertyGroup>
    <Version>1.0.0</Version>
    <Copyright>Copyright ©</Copyright>
    <Company>Prj</Company>
    <Authors>Prj</Authors>
    <Description>Prj</Description>
<!--<TreatWarningsAsErrors>true</TreatWarningsAsErrors>-->
    <WarningsNotAsErrors>NU1901;NU1902;NU1903;NU1904;NU1608;CA2016;CA2201</WarningsNotAsErrors>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <EnableNETAnalyzers>true</EnableNETAnalyzers>
    <AnalysisMode>Recommended</AnalysisMode>
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
    <ServerGarbageCollection>true</ServerGarbageCollection>
    <ConcurrentGarbageCollection>true</ConcurrentGarbageCollection>
    <RetainVM>false</RetainVM>
    <ThreadPoolMinThreads>8</ThreadPoolMinThreads>
    <TieredCompilation>true</TieredCompilation>
    <TieredPGO>true</TieredPGO>
    <PublishTrimmed>false</PublishTrimmed>
  </PropertyGroup>

  <ItemGroup>
    <AdditionalFiles Include="../../BannedSymbols.txt"/>
  </ItemGroup>
</Project>
```

تنظیمات بیس برای استفاده در زمان بیلد هر پروژه
`Directory.Build.targets`
```
<Project>
  <PropertyGroup Condition="'$(PublishAot)' == 'true'">
    <!-- EventSource is disabled by default for non-ASP.NET AOT'd apps, re-enable it -->
    <EventSourceSupport>true</EventSourceSupport>

    <!-- Ensure individual warnings are shown when publishing -->
    <TrimmerSingleWarn>false</TrimmerSingleWarn>
    <!-- But ignore the single warn files marked below to suppress their known warnings. -->
    <NoWarn>$(NoWarn);IL2104</NoWarn>
  </PropertyGroup>

  <Target Name="ConfigureTrimming" BeforeTargets="PrepareForILLink">
    <!-- Single warn the following assemblies, which have known warnings, so the warnings can be suppressed for now. -->
    <ItemGroup>
      <!-- https://github.com/rabbitmq/rabbitmq-dotnet-client/issues/1410 is tracking fixing the one EventSource warning in RabbitMQ. -->
      <IlcArg Include="--singlewarnassembly:RabbitMQ.Client"/>
    </ItemGroup>
  </Target>
</Project>
```

مرکزی کردن ورژن پکیج‌های استفاده شده در پروژه
`Directory.Packages.props`
```
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
    <CentralPackageTransitivePinningEnabled>true</CentralPackageTransitivePinningEnabled>
    <EntityFrameworkCore>7.0.0</EntityFrameworkCore>
  </PropertyGroup>
  <ItemGroup>

  </ItemGroup>
</Project>
```

تنظیمات سرورهای نوگت برای دریافت آنها
`nuget.config`
```
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <add key="NuGetOrg" value="https://api.nuget.org/v3/index.json" />
  </packageSources>
</configuration>

```

قالبی که در زمان پول ریکوست به پروژه به افراد نشان داده می‌شود
`pull_request_template.md`
```
<!--- Provide a general summary of your changes in the Title above -->

## Description
<!--- Describe your changes in detail -->

## Motivation and Context
<!--- Why is this change required? What problem does it solve? -->
<!--- If it fixes an open issue, please link to the issue here. -->

## How Has This Been Tested?
<!--- Please describe in detail how you tested your changes. -->
<!--- Include details of your testing environment, and the tests you ran to -->
<!--- see how your change affects other areas of the code, etc. -->

## Types of changes
<!--- What types of changes does your code introduce? Put an `x` in all the boxes that apply: -->
- [ ] Bug fix (non-breaking change which fixes an issue)
- [ ] New feature (non-breaking change which adds functionality)
- [ ] Breaking change (fix or feature that would cause existing functionality to change)

## Checklist:
<!--- Go over all the following points, and put an `x` in all the boxes that apply. -->
<!--- If you're unsure about any of these, don't hesitate to ask. We're here to help! -->
- [ ] My code follows the code style of this project.
- [ ] My change requires a change to the documentation.
- [ ] I have updated the documentation accordingly.
- [ ] I have read the **CONTRIBUTING** document.
- [ ] I have added tests to cover my changes.
- [ ] All new and existing tests passed.
```

تنظیمات بعد از بیلد در زمان اجرا برنامه. بخشی از آنها در همان فایل csproj بیس قرار دارد
`runtimeconfig.template.json`
```{
  "runtimeOptions": {
    "configProperties": {
      "System.GC.HeapHardLimitPercent": 60
    }
  }
}
```

کافینگ برای تست‌ها مخصوصا IntegrationTest تا جلوی خطا همزمان ران شدن تست‌ها گرفته شود.
`xunit.runner.json`
```
{
  "$schema": "https://xunit.net/schema/current/xunit.runner.schema.json",
  "parallelizeAssembly": false,
  "parallelizeTestCollections": false,
  "maxParallelThreads": 1,
  "diagnosticMessages": true,
  "longRunningTestSeconds": 60,
  "methodDisplay": "classAndMethod",
  "methodDisplayOptions": "all",
  "preEnumerateTheories": true,
  "failSkips": false,
  "stopOnFail": true
}
```
