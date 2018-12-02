---
layout: post
title:  "Dynamic Forms in C# MVC"
date:   2018-12-02 11:42:42 -0500
tags: [C#, MVC, dynamic forms]
visible: 1
permalink: /dynamic-form-data/
---

* TOC
{:toc}

Something that I haven't found well documented is how to generate dynamic form content
in C# MVC without using another tool or framework like Angular, Knockout, etc. It is
entirely possible to generate dynamic form content in C# MVC without the use of third
party tools (other than JQuery!).

### View Models

Our example contains two view models that correspond to the parent form and a collection
of form sections:

The ```ComplicatedFormViewModel``` represents the parent form, with some metadata
properties (e.g. ```Name```) and a collection of form sections, ```FormSections```:

``` csharp
public class ComplicatedFormViewModel
{
    public ComplicatedFormViewModel()
    {
        FormSections = new List<FormSection>();
    }

    public string Name { get; set; }
    public string Description { get; set; }
    public IEnumerable<FormSection> FormSections { get; set; }
}
```

The ```FormSection``` view model represents a smaller subsection of the parent form:

``` csharp
public class FormSection
{
    public string Name { get; set; }
    public string Description { get; set; }
}
```

### Form Binding Conventions

From what I can tell, the default model binder leverages the ```name``` attribute
of HTML elements in order to bind the element's data to a view model property.

For example, the following HTML element is bound as the ```Description``` property of the 
```ComplicatedFormViewModel``` when posted to a controller method:

``` html
<input class="text-box" name="Description" type="text" value="">
```

In the case of nested complex objects, the ```name``` attribute value is delimited
nested property names. Take for example this nested complex object structure:

``` csharp
public class Parent
{
    public Child FirstChild { get; set; }
}

public class Child
{
    public GrandChild AnotherChild { get; set; }
}

public class GrandChild
{
    public string Description { get; set; }
}
```

When posted to a controller method that accepts a ```Parent``` model as its parameter,
the HTML element corresponding to the ```GrandChild.Description``` property would
be named as follows:

``` html
<input class="text-box" name="FirstChild.AnotherChild.Description" type="text" value="">
```

The built-in ```HtmlHelper``` ```EditorFor``` methods take care of this naming
by convention for you when you use them. However, if you call an ```EditorFor``` on
a nested property without the context of its parent(s), the ```name``` attribute will
just be the name of the property, and will not be prefixed by the parent(s).

However, when you use an ```EditorFor``` with the proper parent context, the value
of ```ViewData.TemplateInfo.HtmlFieldPrefix``` is where the ```name``` attribute
comes from. In our case here, its value is ```FirstChild.AnotherChild.Description```.

We need to set the value of ```ViewData.TemplateInfo.HtmlFieldPrefix``` when rendering
the view in order to have the subsection bind correctly via the default model binder.

### Controller Methods

Our example controller contains three methods. The ```Index``` method displays
an editable ```ComplicatedFormViewModel```, which is saved in the ```Create```
method. The ```GetBlankFormSection``` method is used to help generate our
dynamic form content and set the ```ViewData.TemplateInfo.HtmlFieldPrefix ``` value.

``` csharp
public class DynamicFormController : Controller
{
    [HttpGet]
    public ActionResult Index()
    {
        var vm = new ComplicatedFormViewModel();

        return View(vm);
    }

    [HttpPost]
    public ActionResult Create(ComplicatedFormViewModel postedModel)
    {
        // todo: save data, etc...
        return View("Index", postedModel);
    }

    [HttpGet]
    public ActionResult GetBlankFormSection(string prefix, int index)
    {
        var vm = new FormSection();

        ViewData.TemplateInfo.HtmlFieldPrefix = prefix + "[" + index + "]";

        return PartialView(
            "~/Views/DynamicForm/EditorTemplates/FormSection.cshtml",
            vm);
    }
}
```

### Views

Our example has two views configured for filling out the form. One main view displays
the ```ComplicatedFormViewModel```, and we have an editor template for the
```FormSection``` properties.

#### Main Form

The main form is displayed by the ```Index``` method, where we have a button that 
adds a new form section to the bottom of our existing form sections. The button click
is handled in JavaScript.

``` html
@model ViewModels.DynamicForms.ComplicatedFormViewModel

<h2>Index</h2>

@using (Html.BeginForm("Create", "DynamicForm", FormMethod.Post))
{
    <div class="container">
        <div class="row">
            <div class="col">
                @Html.LabelFor(mod => mod.Name)
                @Html.EditorFor(mod => mod.Name)
            </div>
        </div>
        <div class="row">
            <div class="col">
                @Html.LabelFor(mod => mod.Description)
                @Html.EditorFor(mod => mod.Description)
            </div>
        </div>
        <div class="container">
            <div class="row">
                <div class="col">
                    <input type="button"
                           id="addFormSectionBtn"
                           data-prefix="@nameof(Model.FormSections)"
                           value="Add Form Section" 
                           class="btn btn-sm btn-primary"/>
                </div>
            </div>
            <div class="container" id="formSectionTarget">
                @if (Model.FormSections.Any())
                {
                    @Html.EditorFor(mod => mod.FormSections)
                }
                else
                {
                    <div class="row noFormSectionsDisplay">
                        <div class="col">
                            No Form Sections Yet Configured
                        </div>
                    </div>
                }
            </div>
        </div>
        <div class="row">
            <div class="col">
                <input type="submit" class="btn btn-primary" value="Submit" />
            </div>
        </div>
    </div>
}
```

#### Editor Template

The editor template for the ```FormSection``` is located in the
```Views\DynamicForm\EditorTemplates``` folder.

``` html
@model ViewModels.DynamicForms.FormSection

<div class="container formSection">
    <div class="row">
        <div class="col-md-2">
            @Html.LabelFor(mod => mod.Name)
        </div>
        <div class="col-md-10">
            @Html.EditorFor(mod => mod.Name)
        </div>
    </div>
    <div class="row">
        <div class="col-md-2">
            @Html.LabelFor(mod => mod.Description)
        </div>
        <div class="col-md-10">
            @Html.EditorFor(mod => mod.Description)
        </div>
    </div>
</div>
```

### JavaScript

Our JavaScript button handler looks at the data attributes for the button itself,
counts the number of existing form sections, calls into the server to get a new
partial view, and then appends that partial view to the list of existing form sections.

``` javascript
$('#addFormSectionBtn').on('click', function(e){
    e.preventDefault();

    var numberExistingFormSections = $('.formSection').length;
    var modelPrefix = $(this).data('prefix');

    $.ajax({
        url: '/DynamicForm/GetBlankFormSection',
        type: 'GET',
        data: {
            prefix: modelPrefix,
            index: numberExistingFormSections
        },
        success: function (result) {
            $('.noFormSectionsDisplay').hide();
            $('#formSectionTarget').append(result);
        }
    });
});
```

### Rendered HTML

The result of our call to get the ```FormSection``` partial is as follows:

``` html
<div class="container formSection">
    <div class="row">
        <div class="col-md-2">
            <label for="FormSections_0__Name">Name</label>
        </div>
        <div class="col-md-10">
            <input class="text-box single-line" id="FormSections_0__Name" name="FormSections[0].Name" type="text" value="">
        </div>
    </div>
    <div class="row">
        <div class="col-md-2">
            <label for="FormSections_0__Description">Description</label>
        </div>
        <div class="col-md-10">
            <input class="text-box single-line" id="FormSections_0__Description" name="FormSections[0].Description" type="text" value="">
        </div>
    </div>
</div>
```

We have the ```name``` now properly constructed to use the prefix ```FormSections```,
and we update the index based on the number of already existing ```.formSection``` divs.

This will now all post properly with the rest of the form data when submitted
to the ```Create``` method, automatically bound to a ```ComplicatedFormViewModel```
object.