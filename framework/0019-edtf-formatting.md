- Start Date: 2020-03-24
- RFC PR: [#19](https://github.com/inveniosoftware/rfcs/pull/19)
- Authors: Lars Holm Nielsen
- State: DRAFT

# Localization of Extended Date Time Format (EDTF)

## Summary

EDTF allows you to record imprecise dates. This RFCs deals mainly with the
localization of imprecise dates.

## Motivation

There are many use cases where precise dates of records that Invenio
repositories will store are not known. Examples of this could be a photo taken
at some point during World War II, or a journal article publish in August 1858.

The [Extended Date Time Format (EDTF) Spec](https://www.loc.gov/standards/datetime/) defines how these imprecise dates
can be represented.

We want Invenio to be able to a) **store** EDTF dates and date intervals, and
we want to be able to b) **localize** these EDTF dates correctly for end users.

## Detailed design

EDTF dates can be stored as strings according to the EDTF Spec. If search is needed, an earliest/latest date can be calculated from the EDTF date to use for searching.

The primary job of Invenio is to localize the EDTF dates which is not handled by the EDTF library.

### EDTF Level 0 Support

We propose to only support EDTF level 0 dates and dates intervals (i.e.
excluding times and thus not the full EDTF level 0).

Examples of supported EDTF strings:

```
1985-04-12
1985-04
1985
1964/2008
2004-06/2006-08
2004-02-01/2005-02
2004-02-01/2005
2005/2006-02
```

See [EDTF Spec](https://www.loc.gov/standards/datetime/) for further details.

#### Python support

Parsing and validating EDTF strings can be done using the [python-edtf](https://pypi.org/project/edtf/) library. The EDTF parser returns an instance of a subclass of ``EDTFObject``.

### Localization (L10N)

Currently Python date objects are formatted by Flask-BableEx using Babel's
formatting capabilities.

In short date formatting works via the ``format_date`` function:

```python
from flask_babelex import format_date
from datetime import date
format_date(date(2020,3,24), format='long')
```

#### Four formats per locale

The ``format`` parameter, choses between a ``short``, ``medium``, ``full`` and ``long`` format, which is defined for each locale.

For instance:

```python
>>> format_date(format='short')
'3/24/20'
>>> format_date(format='medium')
'Mar 24, 2020'
>>> format_date(format='long')
'March 24, 2020'
>>> format_date(format='full')
'Tuesday, March 24, 2020'
```

This means for each locale (e.g. ``en``, ``da``, ...), there are four different formats defined for a normal date object.


#### Formats for EDTF dates

An EDTF date (i.e. not date interval) can in have:

- 1. a year
- 2. a year and a month
- 3. a year, a month and a day (equal to a Python date)

This means in total we need 8 extra date format patterns for the four formats to localize an EDTF date:

- short: year
- short: year-month
- medium: year
- medium: year-month
- long: year
- long: year-month
- full: year
- full: year-month

Examples of the formatting of ``2020``:

```
short:  2020
medium: 2020
long:   2020
full:   2020
```

Examples of the formatting of ``2020-03``:

```
short:  3/2020
medium: Mar, 2020
long:   March, 2020
full:   March, 2020
```

Likely we can reduce the 8 formats to 4 formats, in that the year formatting can be reduced to one format, and year-month formatting can be reduce to 3 (long/full being equal).

Thus to avoid potentially problems with locales we could just as default provide 8 formats, and fallback in case a format is not defined.


#### Formats for EDTF date intervals

Date intervals are tricky because they can be formatted in multiple ways. Consider e.g. the interval ``2020-02/2020-03``. This interval could be formatted as below:

```
Naive: Feb 2020 - Mar 2020
Compact: Feb-Mar 2020
```

**Naive approach**
The first longer format is a single format, that can format the start/end EDTF date individually and merge them with an interval format.

**Compact approach**
The second compact format, requires an algorithm capable of determinig the *greatest difference* between a start and end date. For more information see [Common Locale Data Repository](https://www.unicode.org/reports/tr35/tr35-dates.html#intervalFormats)

**Naive vs compact**
As an initial implementation we could go with the naive approach, as long as, we can later implement the compact approach. In the naive approach we rely on an already defined fallback pattern per locale (used in ``babel.dates.format_interval``), and thus we don't need any further formats.


### Python API

We suggest to implement a new method ``format_edtf()`` that can format both EDTF date and date intervals.

Examples:

```python
>>> d = parse_edtf('1939-09/1945')
>>> format_edtf(d, format='medium', locale='en')
Sep, 1939 - 1945
```


```python
>>> d = parse_edtf('1939-09')
>>> format_edtf(d, format='medium', locale='en')
Sep, 1939
```

```python
>>> d = parse_edtf('1939-09-01')
>>> format_edtf(d, format='medium', locale='en')
Sep 1, 1939
```

Format EDTF should also allow Python date objects (by simply falling back to ``format_date``):

```python
>>> d = date(1939, 9, 1)
>>> format_edtf(d, format='medium', locale='en')
Sep 1, 1939
```

**Flask and Babel**

We will need two ``format_edtf`` functions. One pure utility function, and one which is Flask request context aware and able to determine use the current locale.

#### Jinja template filters

We also need a Jinja template filter to format dates:

```
{{my_edtfdate|edtfformat(format='long')}}
```

As well as a converter filter:


```
{{my_edtfdate_str|toedtf|edtfformat(format='long')}}
```

#### Marshmallow validation

A Marshmallow field that uses the EDTF library needs to be built that are able to validate and parse EDTF dates. The field could possibly allow to specify if:

- dates are allowed and/or
- date intervals are allowed.

#### Implementation

We recommend implementing the pure utility function in a new module Babel-EDTF. The remaining part can be implemented in Invenio-I18N. This ensures that the core formatting capabilities are usable independently of Invenio.

### Special considerations

#### REST API

We propose to create a Marshmallow field that can dump a localised EDTF date so that a client application will not need to know how to format an EDTF date, but can simply rely on the REST API to do it for them. This way we also don't need to build the formatting for the client-side.

#### Specifying formats

The main issue to address is how the extra formats needed for EDTF needs to be specified. Ideally, Babel-EDTF should be distributed with locale data that could be used.

Babel provides this feaure via the locale class:

```
>>> from babel import Locale
>>> locale = Locale('en', 'US')
>>> locale.date_formats['short']
<DateTimePattern 'M/d/yy'>
```

Thus one could consider also to patch it into this class.

```
>>> locale.edtf_formats['medium']
```

Also, we could have a fallback algorithm that simple takes an existing date format pattern and removes the day and/or month part of the pattern.

## How we teach this

Babel-EDTF needs it's own documentation. In addition, the formatting should be documented in Invenio-I18N.

A special how-to section for Invenio main documentation could be made on modelling imprecise dates.

## Drawbacks

Imprecise dates are important for the library and information science community and handling the properly is important. For people who don't need EDTF dates, we don't see this RFC adds any complexity. The primary drawback is cost of the implementation. Once done, we however don't think the modules will require much maintenance.

## Alternatives

The main alternative, is to simply define a pattern and say if a month or day is not known you set it to January and the 1st of the month. This however is not able to model intervals, and a end-user may not know if e.g. something was published on January 1st of just in the same year. This has been working so far in Zenodo, but there's been a significant amount of request to allow imprecise dates.

The second alternative, is not to do any localization of EDTF dates.

## Unresolved questions

- Specifying formats: We need further research on how formats should be
  specified and distributed per locale - i.e. how do we provide out of the box
  formats, and how can you override these.
- Module design needs to be confirmed.
- Deposit form widgets. This we will address in a separate RFC.
