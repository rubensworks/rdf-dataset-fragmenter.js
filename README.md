# RDF Dataset Fragmenter

[![Build Status](https://travis-ci.com/rubensworks/rdf-dataset-fragmenter.js.svg?branch=master)](https://travis-ci.com/rubensworks/rdf-dataset-fragmenter.js)
[![Coverage Status](https://coveralls.io/repos/github/rubensworks/rdf-dataset-fragmenter.js/badge.svg?branch=master)](https://coveralls.io/github/rubensworks/rdf-dataset-fragmenter.js?branch=master)
[![npm version](https://badge.fury.io/js/rdf-dataset-fragmenter.svg)](https://www.npmjs.com/package/rdf-dataset-fragmenter)

This tool takes one or more datasets as input,
and fragments it into multiple smaller datasets as output,
based on the selected fragmentation strategy.

This reads and writes datasets in a streaming manner,
to support datasets larger than memory.

## Installation

```bash
$ npm install -g rdf-dataset-fragmenter
```
or
```bash
$ yarn global add rdf-dataset-fragmenter
```

## Usage

### Invoke from the command line

This tool can be used on the command line as `rdf-dataset-fragmenter`,
which takes as single parameter the path to a config file:

```bash
$ rdf-dataset-fragmenter path/to/config.json
```

### Config file

The config file that should be passed to the command line tool has the following JSON structure:

```json
{
  "@context": "https://linkedsoftwaredependencies.org/bundles/npm/rdf-dataset-fragmenter/^1.0.0/components/context.jsonld",
  "@id": "urn:rdf-dataset-fragmenter:default",
  "@type": "Fragmenter",
  "Fragmenter:_options_quadSource": {
    "@type": "QuadSourceFile",
    "QuadSourceFile:_options_filePath": "path/to/dataset.ttl"
  },
  "Fragmenter:_options_fragmentationStrategy": {
    "@type": "FragmentationStrategySubject"
  },
  "Fragmenter:_options_quadSink": {
    "@type": "QuadSinkFile",
    "QuadSinkFile:_options_log": true,
    "QuadSinkFile:_options_outputFormat": "application/n-quads",
    "QuadSinkFile:_options_iriToPath": [
      {
        "QuadSinkFile:_options_iriToPath_key": "http://example.org/base/",
        "QuadSinkFile:_options_iriToPath_value": "output/base/"
      },
      {
        "QuadSinkFile:_options_iriToPath_key": "http://example.org/other/",
        "QuadSinkFile:_options_iriToPath_value": "output/other/"
      }
    ]
  }
}
```

The important parts in this config file are:
* `"Fragmenter:_options_quadSource"`: The source from which RDF triples/quads should be read from.
* `"Fragmenter:_options_fragmentationStrategy"`: The strategy that will be employed for fragmentation.
* `"Fragmenter:_options_quadSink"`: The target into which fragmented RDF triples/quads will be written from.
* `"Fragmenter:_options_quadSource"`: Optional transformations over the quad stream.

In this example, the config file will read from the `"path/to/dataset.ttl"` file,
employ subject-based fragmentation, and will write into files in the `"output/"` directory.
For example, the triple `<http://example.org/base/ex1> a <ex:thing>` will be saved into the file `output/base/ex1`,
while the triple `<http://example.org/other/ex2> a <ex:thing>` will be saved into the file `output/other/ex2`.

The available configuration components will be explained in more detail hereafter.

## Configure

### Quad Sources

A quad source is able to provide a stream of quads as input to the fragmentation process.

#### File Quad Source

A file quad source takes as parameter the path to a local RDF file.

```json
{
  "Fragmenter:_options_quadSource": {
    "@type": "QuadSourceFile",
    "QuadSourceFile:_options_filePath": "path/to/dataset.ttl"
  }
}
```

#### Composite Quad Source

A composite quad source allows you to read from multiple quad sources in parallel.

```json
{
  "Fragmenter:_options_quadSource": {
    "@type": "QuadSourceComposite",
    "QuadSourceComposite:_sources": [
      {
        "@type": "QuadSourceFile",
        "QuadSourceFile:_options_filePath": "path/to/dataset1.ttl"
      },
      {
        "@type": "QuadSourceFile",
        "QuadSourceFile:_options_filePath": "path/to/dataset2.ttl"
      },
      {
        "@type": "QuadSourceFile",
        "QuadSourceFile:_options_filePath": "path/to/dataset3.ttl"
      }
    ]
  }
}
```

### Fragmentation Strategy

A fragmentation strategy that fragments a stream of quads into different documents.
Concretely, it takes quads from the source, and pipes them into a quad sink.

#### Subject Fragmentation Strategy

A fragmentation strategy that places quads into their subject's document.

```json
{
  "Fragmenter:_options_fragmentationStrategy": {
    "@type": "FragmentationStrategySubject"
  }
}
```

Optionally, the `FragmentationStrategySubject:_relativePath` property can be used
to define a relative IRI that should be applied to the subject IRI before determining its document.
This will not change the quad, only the document IRI.

#### Object Fragmentation Strategy

A fragmentation strategy that places quads into their object's document.

```json
{
  "Fragmenter:_options_fragmentationStrategy": {
    "@type": "FragmentationStrategyObject"
  }
}
```

#### Composite Fragmentation Strategy

A fragmentation strategy that combines multiple strategies.
This means that all the given strategies will be executed in parallel.

```json
{
  "Fragmenter:_options_fragmentationStrategy": {
    "@type": "FragmentationStrategyComposite",
    "FragmentationStrategyComposite:_strategies": [
      { "@type": "FragmentationStrategySubject" },
      { "@type": "FragmentationStrategyObject" }
    ]
  }
}
```

#### Resource Object Fragmentation Strategy

A fragmentation strategy that groups triples by (subject) resources,
and places quads into the document identified by the given predicate value.

Blank nodes are not supported.

```json
{
  "Fragmenter:_options_fragmentationStrategy": {
    "@type": "FragmentationStrategyResourceObject",
    "FragmentationStrategyResourceObject:_targetPredicateRegex": "vocabulary/hasMaliciousCreator$"
  }
}
```

#### Exception Fragmentation Strategy

A fragmentation strategy that delegates quads to a base strategy,
but allows defining exceptions that should be delegated to other strategies.
These exceptions are defined in terms of a matcher (e.g. match by quad predicate).

The following config uses the subject-based strategy for everything,
except for predicate1 and predicate2 that will be delegated to the object-based strategy.

```json
{
  "Fragmenter:_options_fragmentationStrategy": {
    "@type": "FragmentationStrategyException",
    "FragmentationStrategyException:_strategy": {
      "@type": "FragmentationStrategySubject"
    },
    "FragmentationStrategyException:_exceptions": [
      {
        "@type": "FragmentationStrategyExceptionEntry",
        "FragmentationStrategyExceptionEntry:_matcher": {
          "@type": "QuadMatcherPredicate",
          "QuadMatcherPredicate:_predicateRegex": "vocabulary/predicate1"
        },
        "FragmentationStrategyExceptionEntry:_strategy": {
          "@type": "FragmentationStrategyObject"
        }
      },
      {
        "@type": "FragmentationStrategyExceptionEntry",
        "FragmentationStrategyExceptionEntry:_matcher": {
          "@type": "QuadMatcherPredicate",
          "QuadMatcherPredicate:_predicateRegex": "vocabulary/predicate2"
        },
        "FragmentationStrategyExceptionEntry:_strategy": {
          "@type": "FragmentationStrategyObject"
        }
      }
    ]
  }
}
```

### Quad Sinks

A quad sink is able to direct a stream of quads as output from the fragmentation process.

#### File Quad Sink

A quad sink that writes to files using an IRI to local file system path mapping.

```json
{
  "Fragmenter:_options_quadSink": {
    "@type": "QuadSinkFile",
    "QuadSinkFile:_options_log": true,
    "QuadSinkFile:_options_outputFormat": "application/n-quads",
    "QuadSinkFile:_options_fileExtension": "$.nq",
    "QuadSinkFile:_options_iriToPath": [
      {
        "QuadSinkFile:_options_iriToPath_key": "http://example.org/base/",
        "QuadSinkFile:_options_iriToPath_value": "output/base/"
      },
      {
        "QuadSinkFile:_options_iriToPath_key": "http://example.org/other/",
        "QuadSinkFile:_options_iriToPath_value": "output/other/"
      }
    ]
  }
}
```

Options:
* `"QuadSinkFile:_options_log"`: If a quad counter should be shown to show the current progress.
* `"QuadSinkFile:_options_outputFormat"`: The desired output serialization. (Only `"application/n-quads"` is considered stable at the moment).
* `"QuadSinkFile:_options_fileExtension"`: An optional extension to add to resulting files.
* `"QuadSinkFile:_options_iriToPath"`: A collection of mappings that indicate what URL patterns should be translated into what folder structure.

### Quad Transformers

__Optional__

A quad transformer can transform a stream of quads into another stream of quads.

#### Set IRI Extension Quad Transformer

A quad transformer that enforces the configured extension on all named nodes.

```json
{
  "Fragmenter:_options_transformers": [
    {
      "@type": "QuadTransformerSetIriExtension",
      "QuadTransformerSetIriExtension:_extension": "nq",
      "QuadTransformerSetIriExtension:_iriPattern": "^http://dbpedia.org"
    }
  ]
}
```

Options:
* `"QuadTransformerSetIriExtension:_extension"`: The extension to set, excluding `.`.
* `"QuadTransformerSetIriExtension:_iriPattern"`: An optional regex that to indicate what IRIs this transformer should be applied to. If undefined, all IRIs will be matched.

#### Replace IRI Quad Transformer

A quad transformer that that replaces (parts of) IRIs.

```json
{
  "Fragmenter:_options_transformers": [
    {
      "@type": "QuadTransformerReplaceIri",
      "QuadTransformerReplaceIri:_searchRegex": "^http://www.ldbc.eu",
      "QuadTransformerReplaceIri:_replacementString": "http://localhost:3000/www.ldbc.eu"
    }
  ]
}
```

Options:
* `"QuadTransformerReplaceIri:_searchRegex"`: The regex to search for.
* `"QuadTransformerReplaceIri:_replacementString"`: The string to replace.

#### Remap Resource Identifier Transformer

A quad transformer that matches all resources of the given type,
and rewrites its (subject) IRI (across all triples) so that it becomes part of the targeted resource.

For example, a transformer matching on type `Post` for identifier predicate `hasId` and target predicate `hasCreator`
will modify all post IRIs to become a hash-based IRI inside the object IRI of `hasCreator`.
Concretely, `<ex:post1> a <Post>. <ex:post1> <hasId> '1'. <ex:post1> <hasCreator> <urn:person1>`
will become
`<urn:person1#Post1> a <Post>. <urn:person1#Post1> <hasId> '1'. <urn:person1#post1> <hasCreator> <urn:person1>`.

**WARNING:** This transformer assumes that all the applicable resources
have `rdf:type` occurring as first triple with the resource IRI as subject.

```json
{
  "Fragmenter:_options_transformers": [
    {
      "@type": "QuadTransformerRemapResourceIdentifier",
      "QuadTransformerRemapResourceIdentifier:_newIdentifierSeparator": "#Post",
      "QuadTransformerRemapResourceIdentifier:_typeRegex": "vocabulary/Post$",
      "QuadTransformerRemapResourceIdentifier:_identifierPredicateRegex": "vocabulary/id$",
      "QuadTransformerRemapResourceIdentifier:_targetPredicateRegex": "vocabulary/hasCreator$"
    }
  ]
}
```

Options:
* `"QuadTransformerRemapResourceIdentifier:_newIdentifierSeparator"`: Separator string to use inbetween the target IRI and the identifier value when minting a new resource IRI.
* `"QuadTransformerRemapResourceIdentifier:_typeRegex"`: The RDF type that should be used to capture resources.
* `"QuadTransformerRemapResourceIdentifier:_identifierPredicateRegex"`: Predicate regex that contains a resource identifier.
* `"QuadTransformerRemapResourceIdentifier:_targetPredicateRegex"`: Predicate regex that contains an IRI onto which the resource identifier should be remapped.

#### Append Resource Link Transformer

A quad transformer that matches all resources of the given type, and appends a link.

```json
{
  "Fragmenter:_options_transformers": [
    {
      "@type": "QuadTransformerAppendResourceLink",
      "QuadTransformerAppendResourceLink:_typeRegex": "vocabulary/Person$",
      "QuadTransformerAppendResourceLink:_predicate": "http://example.org/postsIndex",
      "QuadTransformerAppendResourceLink:_link": "/posts"
    }
  ]
}
```

Options:
* `"QuadTransformerAppendResourceLink:_typeRegex"`: The RDF type that should be used to capture resources.
* `"QuadTransformerAppendResourceLink:_predicate"`: Predicate IRI to define the link.
* `"QuadTransformerAppendResourceLink:_link"`: The relative link from the resource identifier.
* `"QuadTransformerAppendResourceLink:_linkType"`: Optional: `rdf:type` IRI that should be assigned to the link IRI as an extra triple.
* `"QuadTransformerAppendResourceLink:_reverse"`: Optional: If the subject and object of the link triple should be revered.
* `"QuadTransformerAppendResourceLink:_linkRemoveTrailingSlash"`: Optional: If trailing slashes should be forcefully removed from the link IRI.

#### Append Resource SCL Transformer

A quad transformer that matches all resources of the given type,
and appends an ACL policy using the `scl:appliesTo` and `scl:scope` predicates.

Example output:
```turtle
<http://example.org/person> a <http://example.org/vocabulary/Person>.
<http://example.org/person#policy-posts> scl:appliesTo <http://example.org/person>;
                                         scl:scope "---MY POLICY---".
```

```json
{
  "Fragmenter:_options_transformers": [
    {
      "@type": "QuadTransformerAppendResourceScl",
      "QuadTransformerAppendResourceScl:_typeRegex": "vocabulary/Person$",
      "QuadTransformerAppendResourceScl:_identifierSuffix": "#policy-posts",
      "QuadTransformerAppendResourceScl:_sclPolicy": "FOLLOW ?posts { <> <http://www.w3.org/1999/02/22-rdf-syntax-ns#seeAlso> ?posts }"
    }
  ]
}
```

Options:
* `"QuadTransformerAppendResourceScl:_typeRegex"`: The RDF type that should be used to capture resources.
* `"QuadTransformerAppendResourceScl:_identifierSuffix"`: String to append to the resource IRI to mint the policy IRI.
* `"QuadTransformerAppendResourceScl:_sclPolicy"`: The SCL policy to append.

### Quad Matchers

Different strategies for matching quads.
These matchers are mainly used for a `FragmentationStrategyExceptionEntry`.

#### Predicate Matcher

Matches a quad by the given predicate regex.

```json
{
  "Fragmenter:_options_fragmentationStrategy": {
    "@type": "FragmentationStrategyException",
    "FragmentationStrategyException:_strategy": {
      "@type": "FragmentationStrategySubject"
    },
    "FragmentationStrategyException:_exceptions": [
      {
        "@type": "FragmentationStrategyExceptionEntry",
        "FragmentationStrategyExceptionEntry:_matcher": {
          "@type": "QuadMatcherPredicate",
          "QuadMatcherPredicate:_predicateRegex": "vocabulary/predicate1"
        },
        "FragmentationStrategyExceptionEntry:_strategy": {
          "@type": "FragmentationStrategyObject"
        }
      }
    ]
  }
}
```

#### Resource Type Matcher

A quad matcher that matches all resources of the given type.

Blank nodes are not supported.

**WARNING:** This matcher assumes that all the applicable resources
have `rdf:type` occurring as first triple with the resource IRI as subject.

```json
{
  "Fragmenter:_options_fragmentationStrategy": {
    "@type": "FragmentationStrategyException",
    "FragmentationStrategyException:_strategy": {
      "@type": "FragmentationStrategySubject"
    },
    "FragmentationStrategyException:_exceptions": [
      {
        "@type": "FragmentationStrategyExceptionEntry",
        "FragmentationStrategyExceptionEntry:_matcher": {
          "@type": "QuadMatcherResourceType",
          "QuadMatcherResourceType:_typeRegex": "vocabulary/Person$"
        },
        "FragmentationStrategyExceptionEntry:_strategy": {
          "@type": "FragmentationStrategyResourceObject",
          "FragmentationStrategyResourceObject:_targetPredicateRegex": "vocabulary/hasMaliciousCreator$"
        }
      }
    ]
  }
}
```

## Extend

This tool has been created with extensibility in mind.
After forking+cloning this repo and running `npm install` or `yarn install`,
you can create new components inside the `lib/` directory.

The following TypeScript interfaces are available for implementing new components:
* `IQuadSource`: A quad source is able to provide a stream of quads.
* `IFragmentationStrategy`: A fragmentation strategy that fragments quads into different documents.
* `IQuadSink`: A quad sink is able to consume quads to document IRI targets.

If you want to use your newly created component, make sure to execute `npm run build` or `yarn run build`.
After that, you can include your components in your config file by referring to them from `@type` with their class name.

## License
This software is written by [Ruben Taelman](http://rubensworks.net/).

This code is released under the [MIT license](http://opensource.org/licenses/MIT).
