---
title: "Friendly Python Clojures in an automatic Text Annotator"
date: 2021-11-06T02:53:16+02:00
draft: false
---

While creating some pipelines for automatic text annotation, I encountered a bug that made me realise that I didn't really get the gist of a basic block of programming.

It's closures. They're there thanks to your language having first-class functions and you may have encountered them in academia or some programming tutorial, even used them in some toy examples. I know I had. Closures in a sandbox are straight forward. Still, you may not know their power until you'd seen them in the wild.

In Python at least, you won't need to know closures to solve most simple business problems. But once a more sofisticated system is in place and you won't want to change parts of the system (in many chases it's not practical due to the parts being held in out-of-your-control packages), sometimes a closure comes in handy for providig data encapsulation, feels idiomatic and will reduce the mental overload you'd have to deal with when using a more complex structure, such as a class.


Still here? Then I'll reconstruct some [SpaCy](https://spacy.io/) and [skweak](https://github.com/NorskRegnesentral/skweak) functionalities for you so you'll better understant when to use these closures.

## SpaCy objects

[SpaCy](https://spacy.io/) is here for you to help you build easy NLP pypelines.
Central to this package are the `Doc` objects (short for document). It's a neatly way to pack data for NLP and if it doesn't provide what you need out of the box, you can [always extend it's functionalities](https://spacy.io/usage/processing-pipelines#custom-components-attributes) to match your usecase.


```python
from typing import Callable, Generator, Iterable, List

class Doc:
    def __init__(self, text) -> None:
        self.text = text
        self.tokens: List[str] = text.split()
        self.spans: Span = []
    def __len__(self):
        return len(self.tokens)
    def __getitem__(self, position):
        return self.tokens[position]
```

By implementing `__len__` and `__getitem__` [double-under functions](https://www.geeksforgeeks.org/dunder-magic-methods-python/) we got the ability to iterate through the Doc's tokens with a simple for as below. This is thanks to the Python datamodel. It's outside the scope of this post, but learning to leverage the datamodel will pay dividends on your effectiveness in Python. The first chapter of [Fluent Python](https://learning.oreilly.com/library/view/fluent-python/9781491946237/) introduces it in the first chapter in a very neat way. If you like video format more, [James Powell](https://www.youtube.com/watch?v=AmHE0kZhLIQ) got you covered.


```python
doc = Doc("Today I ate garbonzo beans")
for token in doc:
    print(token)
```

    Today
    I
    ate
    garbonzo
    beans


A `Span` is a slice of a `Doc`. Usually it can.. span multiple tokens, but today I have a feeling that all the spans we'll look at will match exactly one token. Also, in our case the spans will be always labeled.


```python
from typing import NamedTuple

class Span(NamedTuple):
    position: int
    label: str

```

# skweak functions
If you haven't looked at the [skweak repo](https://github.com/NorskRegnesentral/skweak) yet, it suffices to know that it provides a neat way of composing simple annotators to get a better one.
Now, skweak provides us with some very interesting classes. One is a `FunctionAnnotator`. This takes a function that returns a list of spans from a document and attaches these spans to the given document. 


```python
class FunctionAnnotator:
    """Annotation based on a heuristic function that generates (start,end,label)
    given a spacy document"""
    def __init__(self, function: Callable[[Doc], Iterable[Span]]):
        self.find_spans = function
		
    def __call__(self, doc: Doc) -> Doc:
        # We start by clearing all existing annotations
        doc.spans = []

        for position, label in self.find_spans(doc):
            doc.spans.append(Span(position, label))

        return doc
```

Let's see a simple labeling function we may use


```python
def animal_labeling_function(doc: Doc) -> Generator[Span, None, None]:
    for position, token in enumerate(doc.tokens):
        if token.startswith('a'):
            yield Span(position, 'ANIMAL')

doc = Doc('this animal is some kind of antilope')
animal_annotator = FunctionAnnotator(animal_labeling_function)
doc = animal_annotator(doc)

print(doc.spans)
```

    [Span(position=1, label='ANIMAL'), Span(position=6, label='ANIMAL')]


The `FunctionAnnotatorAggregator` takes multiple annotator functions and combines them in some fancy way. We won't do it justice with the implementation below.
We'll just make it force our documents to have a maximum of one label per span. We will also sort them by the order of appearance.


```python
class FunctionAnnotatorAggregator:
    """Base aggregator to combine all labelling sources into a single annotation layer"""
    def __init__(self, annotators: Iterable[FunctionAnnotator]) -> None:
        self.annotators = annotators
    
    def __call__(self, doc: Doc) -> Doc:
        spans_dict = dict()
        for annotator in self.annotators:
            for span in annotator(doc).spans:
                spans_dict[span.position] = span.label
        
        
        doc.spans = []
        for position, label in spans_dict.items():
            doc.spans.append(Span(position, label))
        doc.spans.sort()
        
        return doc
```


```python
def verb_labeling_function(doc: Doc) -> Generator[Span, None, None]:
    for position, token in enumerate(doc.tokens):
        if token in ['is', 'has']:
            yield Span(position, 'VERB')

verb_annotator = FunctionAnnotator(verb_labeling_function)

aggregated_annotator = FunctionAnnotatorAggregator([animal_annotator, verb_annotator])

doc = aggregated_annotator(doc)

print(doc.spans)

```

    [Span(position=1, label='ANIMAL'), Span(position=2, label='VERB'), Span(position=6, label='ANIMAL')]


# The problem

All nice and dandy, the packages are well implemented and work as expected! 
Now, since the aggregator is so fancy, we may wish to programatically generate some labeling functions from a list of excellent heuristic parameters
<a id='problem_cell'></a>


```python
heuristic_parameters = [
    ('M', 'MAMMAL'),
    ('F', 'FISH'),
    ('B', 'BIRD')
    ]

labeling_functions = []
for strats_with, label in heuristic_parameters:
    def labeling_function(doc: Doc) -> Generator[Span, None, None]:
        for position, word in enumerate(doc.tokens):
            if word.startswith(strats_with):
                yield Span(position, label)
    labeling_functions += [labeling_function]

strats_with, label = 'B', 'BOVINE'

```

<a id='problem_cell_2'></a>


```python
doc = Doc("Monkeys are red. Firefish are blue; Besra is a bird and so are you")

# we'll define this function since we'll use it a lot more below
def print_spans_from_labelers(doc, labeling_functions):
    annotators = [FunctionAnnotator(labeling_function) for labeling_function in labeling_functions]
    aggregated_annotator = FunctionAnnotatorAggregator(annotators)

    doc = aggregated_annotator(doc)

    print(doc.spans)
print_spans_from_labelers(doc, labeling_functions)

```

    [Span(position=6, label='BOVINE')]


What happened? It seems that only the last function was applied. Let's look at the `labeling_functions`


```python
for labeling_function in labeling_functions:
    print(labeling_function)
```

    <bound method LabelingCallable.labeling_function of <__main__.LabelingCallable object at 0x7fa6981669a0>>
    <bound method LabelingCallable.labeling_function of <__main__.LabelingCallable object at 0x7fa698166d30>>
    <bound method LabelingCallable.labeling_function of <__main__.LabelingCallable object at 0x7fa698166940>>


They point to the same memory address. This is because Python redefines functions in place. 
But this is not the only problem. Let's rewrite this with lambda functions. 

*Note* if you haven't worked with list comprehensions before: don't worry about it; think of the code below as a way to create a new function without replacing the existing function with the same name


```python
labeling_functions = [
                        lambda doc: 
                            ( # this is also a generator in Python; it's the same syntax as list comprehension
                            # but we use round braces instead of square ones
                                Span(position, label) 
                                    for position, word in enumerate(doc.tokens) 
                                    if word.startswith(strats_with)
                            )
                        for strats_with, label in heuristic_parameters
                    ]

for labeling_function in labeling_functions:
    print(labeling_function)
```

    <function <listcomp>.<lambda> at 0x7fa69815f160>
    <function <listcomp>.<lambda> at 0x7fa69815f1f0>
    <function <listcomp>.<lambda> at 0x7fa69815f280>


But when we want to print the function the problem stays.


```python
print_spans_from_labelers(doc, labeling_functions)
```

    [Span(position=6, label='BIRD')]


This is because of scoping. The problem is that, since we didn't declare strats_with, label in the lambda body or parameters, the lambdas will always look in the scope immediately outside them and they will find the last values that `strats_with`, `label` had.
If you come from other languages it might be strange to you, but Python doesn't create a new scope for the `for` body. Instead it uses the same local scope. This is why `strats_with, label = 'B', 'BOVINE'` in snippet [8](#problem_cell) produced snippet [9](#problem_cell_2) to display the label as 'BOVINE'




But be not affraid! There is a solution:


```python
def function_closure(strats_with, label):
    local_strats_with, local_label = strats_with, label
    def labeling_function(doc: Doc) -> Generator[Span, None, None]:
        for position, word in enumerate(doc.tokens):
            if word.startswith(local_strats_with):
                yield Span(position, local_label)
    return labeling_function

labeling_functions = [function_closure(strats_with, label) for strats_with, label in heuristic_parameters]
```

Now, when we get the annotators things go as expected.


```python
print_spans_from_labelers(doc, labeling_functions)
```

    [Span(position=0, label='MAMMAL'), Span(position=3, label='FISH'), Span(position=6, label='BIRD')]


But why is this different from the last attempt? This time, by calling `function_closure` we are creating a local scope around the labeling function and put the `strats_with` and `label` variables in it. These variables are recreated every time we call `function_closure`. It also recreates `labeling_function` since functions are regular objects in Python and different calls can’t trample on one another’s local variables. 

A good mental model is to think of function values as containing both the code in their body and the environment in which they are created.*

*lifted from [*Eloquent Javascript*](https://eloquentjavascript.net/03_functions.html) book

Inspecting the functions will also confirm this: 


```python
print(repr(labeling_functions[0]))
print(type(labeling_functions[0]))
```

    <function function_closure.<locals>.labeling_function at 0x7fa698152ca0>
    <class 'function'>


One improvement we can make is not using `local_strats_with`, `local_label`, since parameters are themselves local variables 


```python
def function_closure(strats_with, label):
    def labeling_function(doc: Doc) -> Generator[Span, None, None]:
        for position, word in enumerate(doc.tokens):
            if word.startswith(strats_with):
                yield Span(position, label)
    return labeling_function

labeling_functions = [function_closure(strats_with, label) for strats_with, label in heuristic_parameters]
print_spans_from_labelers(doc, labeling_functions)
```

    [Span(position=0, label='MAMMAL'), Span(position=3, label='FISH'), Span(position=6, label='BIRD')]


Now compare it with how you'd implement a regular class:


```python
class LabelingCallable:
    def __init__(self, starts_with, label) -> None:
        self.starts_with = starts_with
        self.label = label

    def __call__(self, doc) -> Generator[Span, None, None]:
        for position, word in enumerate(doc.tokens):
            if word.startswith(self.starts_with):
                yield Span(position, self.label)

labeling_functions = [LabelingCallable(strats_with, label) for strats_with, label in heuristic_parameters]
print_spans_from_labelers(doc, labeling_functions)
```

    [Span(position=0, label='MAMMAL'), Span(position=3, label='FISH'), Span(position=6, label='BIRD')]


The object is used only because it can function as a regular function - something that a regular function is more fit to do. This also requires you to come up with a naming convention for this kind of classes. And it doesn't fit with the fact that `skweak` expects a function (as the code and docstrings imply), even if it masquerades as one.

You can also do something like this:


```python
class LabelingCallable:
    def __init__(self, starts_with, label) -> None:
        self.starts_with = starts_with
        self.label = label

    def labeling_function(self, doc) -> Generator[Span, None, None]:
        for position, word in enumerate(doc.tokens):
            if word.startswith(self.starts_with):
                yield Span(position, self.label)

labeling_functions = [LabelingCallable(strats_with, label).labeling_function for strats_with, label in heuristic_parameters]
print_spans_from_labelers(doc, labeling_functions)
print(repr(labeling_functions[0]))
print(type(labeling_functions[0]))
```

    [Span(position=0, label='MAMMAL'), Span(position=3, label='FISH'), Span(position=6, label='BIRD')]
    <bound method LabelingCallable.labeling_function of <__main__.LabelingCallable object at 0x7fa6981669a0>>
    <class 'method'>


Maybe this is something you'll eventually want but, for our intended purposes, this is basically a closure with extra steps. 

# Ending notes

In this post we explored closures, a solution for providing data locality.
We saw one usecase in which they can be helpful and are better suited than classes.

If this is the kind of content you enjoy, let me know!
