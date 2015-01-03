---
layout: post
title: Roslyn - Creating an introduce and initialize field refactoring
description: How to create a introduce and initialize field Roslyn refactoring.
---
Here I'll take a look at how we can create a [Roslyn](https://roslyn.codeplex.com/) code refactoring using [Visual Studio 2015 Preview](http://www.visualstudio.com/en-us/news/vs2015-preview-vs.aspx).

We'll make a code refactoring that introduces and initializes a field, based on a constructor argument. If you've used [Resharper](https://www.jetbrains.com/resharper/) you know what I'm talking about.

## Creating the project ##

Fire up Visual Studio, go to File -> New Project (CTRL + Shift + N) and pick "Code Refactoring (VSIX)" under "Extensibility":

![File, new project](/assets/roslyn-refactoring-new-proj.png)

The newly created solution consists of two projects: 

- The VSIX package project
- The code refactoring project
 
If we look inside the "CodeRefactoringProvider.cs" file of the code refactoring project, we'll see that it includes a complete code refactoring as a starting point. In its simplest form implementing a code refactoring consists of three steps:

- Find the node that's selected by the user
- Check if it's the type of node we're interested in
- Offer our code refactoring if it's right type

We can test/debug the default implementation by simply pressing F5, as with any other type of project. This will start an experimental instance of Visual Studio and deploy our VSIX content. We can then open a project, place our cursor on a class definition and click the light bulb (shortcut: CTRL + .). As we can see in the picture below the "Reverse type name" code refactoring is presented as well as a preview of the changes.

![Code refactoring popup](/assets/roslyn-refactoring-default-popup.png)

## Implementing our refactoring ##

It's worth mentioning the [Roslyn Syntax Analyzer](https://roslyn.codeplex.com/wikipage?title=Syntax%20Visualizer&referringTitle=Home), which is a life saver when trying to wrap your head around syntax trees, syntax nodes, syntax tokens, syntax trivia etc.

Here's the body of our new `ComputeRefactoringsAsync` method, let's go through it:

```csharp
var root = await context.Document
                        .GetSyntaxRootAsync(context.CancellationToken)
                        .ConfigureAwait(false);
var node = root.FindNode(context.Span);
var parameter = node as ParameterSyntax;
if (parameter == null)
{
    return;
}

var parameterName = RoslynHelpers.GetParameterName(parameter);
var underscorePrefix = "_" + parameterName;
var uppercase = parameterName.Substring(0, 1)
                             .ToUpper() + 
							  parameterName.Substring(1);

if (RoslynHelpers.VariableExists(root, parameterName, underscorePrefix, uppercase))
{
    return;
}

var action = CodeAction.Create(
    "Introduce and initialize field '" + underscorePrefix + "'",
    ct => 
    CreateFieldAsync(context, parameter, parameterName, underscorePrefix, ct));

context.RegisterRefactoring(action);

```

As before we're getting the node that's selected by the user, but this time we're only interested if it's a node of type `ParameterSyntax`. I used the Roslyn Syntax Analyzer mentioned earlier to figure out what to look for.

Once we've found the correct node type we extract the parameter name and create different variations of it, to search for existing fields with those names. If the parameter name is "bar", we'll look for that as well as "_bar" and "Bar". 

if none of the variables exist we register our refactoring.

Now for the second and somewhat more tricky part, the actual refactoring logic. [RoslynQouter](https://github.com/KirillOsenkov/RoslynQuoter) is of great help here, since it shows what API calls we need to make to construct the syntax tree for a given program.

Here's the code, we'll go through it below:

```csharp
private async Task<Document> CreateFieldAsync(
           CodeRefactoringContext context, ParameterSyntax parameter,
           string paramName, string fieldName, 
           CancellationToken cancellationToken)
{
    var oldConstructor = parameter
                         .Ancestors()
                         .OfType<ConstructorDeclarationSyntax>()
                         .First();
    var newConstructor = oldConstructor
                         .WithBody(oldConstructor.Body.AddStatements(
                          SyntaxFactory.ExpressionStatement(
                          SyntaxFactory.AssignmentExpression(
                          SyntaxKind.SimpleAssignmentExpression,
                          SyntaxFactory.IdentifierName(fieldName),
                          SyntaxFactory.IdentifierName(paramName)))));

    var oldClass = parameter
                   .FirstAncestorOrSelf<ClassDeclarationSyntax>();
    var oldClassWithNewCtor = 
                 oldClass.ReplaceNode(oldConstructor, newConstructor);

    var fieldDeclaration = RoslynHelpers.CreateFieldDeclaration(
                RoslynHelpers.GetParameterType(parameter), fieldName);
    var newClass = oldClassWithNewCtor.WithMembers(
                   oldClassWithNewCtor.Members
                                      .Insert(0, fieldDeclaration))
                   .WithAdditionalAnnotations(Formatter.Annotation);

    var oldRoot = await context.Document
                               .GetSyntaxRootAsync(cancellationToken)
                               .ConfigureAwait(false);
    var newRoot = oldRoot.ReplaceNode(oldClass, newClass);

    return context.Document.WithSyntaxRoot(newRoot);
}
```

First we find the original class constructor, which contains our parameter.

We use that and add an assignment expression in the body of the modified constructor. So if the original constructor was like this:

```csharp
public Foo(int bar)
{
}
```

the assignment expression results in this:

```csharp
public Foo(int bar)
{
	_bar = bar;
}
```

We take the old class declaration and create a new one with the constructor replaced.

Then we create the field declaration:

```csharp
private readonly int _bar;
```

Finally we insert the field declaration as the first member in our new class declaration, format our code and return the new document.

## The final result ##

![Code refactoring popup](/assets/roslyn-refactoring-new-popup.png)

The result will not put Resharper out of business any time soon, but I'm happy that it was quite easy to get started :-)

The "RoslynHelpers" class i created could be replaced with extension methods, but checking out Roslyn was my main focus here.

The code is available [here](https://github.com/trydis/roslyn-introduce-and-init-field).