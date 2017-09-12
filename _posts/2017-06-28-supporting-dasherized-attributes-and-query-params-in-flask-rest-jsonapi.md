---
layout: post
title:  "Supporting kebab-case Attributes and Query Params in flask-rest jsonapi"
date:   2017-06-07 18:34:56 +0530
categories: gsoc
---
![Kebab]({{site.baseurl}}/images/kebab.jpg)

In the [Open Event API Server](https://github.com/fossasia/open-event-orga-server) Project, it was decided to dasherize all the attributes of the API.

### What was the need for dasherizing the attributes in the API ?
All the attributes in our database models are seperated by underscores i.e `first name` would be stored as `first_name`. But most of the API client implementations support dasherized attributes by default. In order to attract third party client implementations in the future and making the API easy to setup for them was the primary reason behind this decision. Note: The dasherized version for `first_name` will be `first-name`. Also to quote the official json-api spec recommendation for the same:
> Member names SHOULD contain only the characters “a-z” (U+0061 to U+007A), “0-9” (U+0030 to U+0039), and the hyphen minus (U+002D HYPHEN-MINUS, “-“) as separator between multiple words.

[flask-rest-jsonapi](http://flask-rest-jsonapi.readthedocs.io/en/latest/) is the API framework used by the project. We were able to dasherize the API responses and requests by adding `inflect=dasherize` to each API schema, where `dasherize` iss the following function:
```python
def dasherize(text):
    return text.replace('_', '-')
```

[flask-rest-jsonapi](http://flask-rest-jsonapi.readthedocs.io/en/latest/) also provides powerful features like the following through query params:

- [Sparse fieldsets](http://flask-rest-jsonapi.readthedocs.io/en/latest/sparse_fieldsets.html)
- [Sorting](http://flask-rest-jsonapi.readthedocs.io/en/latest/pagination.html)
- [Include related objects](http://flask-rest-jsonapi.readthedocs.io/en/latest/include_related_objects.html)

But we observed that the query params were not being dasherized which rendered the above awesome features useless :( . The reason for this was that [flask-rest-jsonapi](http://flask-rest-jsonapi.readthedocs.io/en/latest/) took the query params `as-is` and search for them in the API schema. As Python variable names cannot contain a dash, naming the attributes with a dash in the internal API schema was out of the question.

For adding dasherizing support to the query params, change in the `QueryStringManager` located at `querystring.py` of the framework root are required. A config variable named `DASHERIZE_API` was added to turn this feature on and off.

### Following are the changes required for dasherizing query params:

For Sparse Fieldsets in the `fields` function, replace the following line:
```python
result[key] = [value]
```
with
```python
if current_app.config['DASHERIZE_API'] is True:
    result[key] = [value.replace('-', '_')]
else:
    result[key] = [value]
```

For sorting, in the `sorting` function, replace the following line:
```python
field = sort_field.replace('-', '')
```
with
```python
if current_app.config['DASHERIZE_API'] is True:
    field = sort_field[0].replace('-', '') + sort_field[1:].replace('-', '_')
else:
    field = sort_field[0].replace('-', '') + sort_field[1:]
```

For Include related objects, in `include` function, replace the following line:
```python
return include_param.split(',') if include_param else []
```
with
```python
if include_param:
    param_results = []
    for param in include_param.split(','):
        if current_app.config['DASHERIZE_API'] is True:
            param = param.replace('-', '_')
        param_results.append(param)
    return param_results
return []
```
