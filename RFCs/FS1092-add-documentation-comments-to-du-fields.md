# F# RFC FS-1092 - Add XML Documentaion comments to Union Case Fields

## Table of Contents

- [Summary](#summary)
- [Motivation](#motivation)
- [Detailed Design](#detailed-design)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Compatibility](#compatibility)
- [Unresolved Questions](#unresolved-questions)

The design suggestion [Allow for XML documentation comments on DU Case fields](https://github.com/fsharp/fslang-suggestions/issues/948) has been marked "approved in principle".

This RFC covers the detailed proposal for this suggestion.

- [x] Approved in principle
- [x] [Suggestion](https://github.com/fsharp/fslang-suggestions/issues/948)
- [ ] [Implementation](https://github.com/dotnet/fsharp/pull/FILL-ME-IN)
- [ ] Design Review Meeting(s) with @dsyme and others invitees
- [Discussion](https://github.com/fsharp/fslang-design/issues/PLEASE-ADD-A-DISCUSSION-ISSUE-AND-LINK-HERE)

## Summary

Discriminated Union Cases, similarly to records, are stored as classes whose fields are exposed as IL properties, which are allowed to expose XL documentation for consumers. However, unlike records, Union Case fields are currently do not accept XML documentation comments prior to their definition. This represents a gap in capabilities for user-facing documentation for these core data types. This RFC documents the changes related to allowing these XML documentation comments to be applied to Union Case fields.

## Motivation

This aligns the two unique F# data types with each other in terms of documentation.

## Detailed design

The SyntaxTree node for [Union Case fields](https://github.com/dotnet/fsharp/blob/main/src/fsharp/SyntaxTree.fs#L1672-L1675) is a list of `SynField`, which already have a field for [XML Documentation comments](https://github.com/dotnet/fsharp/blob/main/src/fsharp/SyntaxTree.fs#L1719-L1728), so the exposed API surface of the SyntaxTree will not need to change.  Primarily what will change are:

- the [parser rule](https://github.com/dotnet/fsharp/blob/main/src/fsharp/pars.fsy#L2474-L2480) for union case fields,
- the [computation of XmlDocSig](https://github.com/dotnet/fsharp/blob/main/src/fsharp/XmlDocFileWriter.fs#L27-L28) (aka the resolved name for the element in the generated XML documentation file) for the Union Case fields, and
- the [generation of XML documentation elements](https://github.com/dotnet/fsharp/blob/main/src/fsharp/XmlDocFileWriter.fs#L72) for the Union Case fields

### Parser Rule

`SyntaxTreeOps.mkNamedField` and `mkAnonField` will learn a new `XmlDoc.PreXmlDoc` parameter, which will be flowed through to the `SynField.Field` created in them.

The parser rules for `unionCaseReprElement` will both `grabXmlDoc` from the current parser state before the first token in the rule and pass that to the `mkNamedField/mkAnonField` function they then call.

### XmlDocSig computation

During Union Case XmlDocSig generation, the `RecdFieldsArray` of the Union Case will be walked as well, and each field that `hasDoc` will generate an XmlDocSig for an IL Property of the form `<ptext>.<compiled name of parent Type Constructor>.<idText of parent Union Case>.<idText of field>`.

That is, a declaration like

```fsharp
module Container

type Parent = | Case1 of field1: string
```

would result in an XmlDocSig for `field1` of the form `P:Container.Parent.Case1.field1`.

A property is chosen as the XmlDocSig prefix because Union Case fields are exposed as IL properties with getters.

### XML Documentation Generation

During Union Case XML Documentation generation, the `RecdFieldsArray` of the Union Case will be walked as well, and each field that `hasDoc` will be written to the underlying list of members via `addMember`, using its pre-computed `XmlDocSig` and user-provided `XmlDoc`.

## Drawbacks

This will cause larger XML Documentation files.
This will allow for too much clarity in documentation of types, and our users will get spoiled.

## Alternatives

It could be possible to augment Union Case XML documentation with structures specifically oriented towards authoring documentation for the fields of the Union Case, rather than the Union Case itself. This would incur additional post-processing and validation costs that we mostly sidestep if the XML Documentation is simply allowed on each field.

## Compatibility

Please address all necessary compatibility questions:

- Is this a breaking change? No, it allows strictly more forms of source code than were allowed before this change.
- What happens when previous versions of the F# compiler encounter this design addition as source code? Nothing, as current compilers simply throw away this information.
- What happens when previous versions of the F# compiler encounter this design addition in compiled binaries? Nothing, because there has been no change to the API of the SyntaxTree itself.
- If this is a change or extension to FSharp.Core, what happens when previous versions of the F# compiler encounter this construct? N/A

## Unresolved questions

- Is the choice of `P` (meaning Property) for the XmlDocSig correct?
- Is the choice of XmlDocSig segments correct? (enclosing container, parent type name, union case name, field name) 
