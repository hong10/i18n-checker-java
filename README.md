i18n-checker - Java version
============

**Notice:** This project was discontinued in February 2014.

A Java implementation of the [W3C Internationalization Checker](http://validator.w3.org/i18n-checker/). Provides an API for running internationalization (i18n) checks on a document.

[![Build status image from travis-ci.org](https://travis-ci.org/w3c/i18n-checker.png?branch=master)](http://travis-ci.org/w3c/i18n-checker)

Contents:

1. [Usage](#usage)
2. [Developers](#developers)
3. [Testing](#testing)
4. [License](#license)

Usage
-----
To perform an i18n check, import [`org.w3.i18n.I18nChecker`](http://github.com/w3c/i18n-checker/blob/master/src/main/java/org/w3/i18n/I18nChecker.java) and use either:
* `static List<Assertion> check(URL url)`, for online checks; or
* `static List<Assertion> check(URL url, InputStream body, Map<String,List<String>> headers)`, for offline checks.

The results are provided as a list of  [`org.w3.i18n.Assertion`](https://github.com/w3c/i18n-checker/blob/master/src/main/java/org/w3/i18n/Assertion.java). An Assertion has an ID, level, HTML title, HTML description, and a list of contexts. Together these items say something thing about a document that was checked. For example, an Assertion could have:
```
id = charset_http
level = INFO
htmlTitle = Charset declaration in HTTP response header 'Content-Type'
htmlDescription = …
contexts = [utf-8, Content-Type: text/html; charset=utf-8]
```

An Assertion level can be `INFO`, `WARNING`, `ERROR`, or `MESSAGE`. `MESSAGE` level Assertions contain only meta information about the operation of the checker. `WARNING` and `ERROR` level Assertions describe i18n problems with the document. In such cases the `htmlTitle` and `htmlDescription` provide advice on solving the problem. `INFO` Assertions contain useful information, relevant to the i18n of document (such as a list of the language and charset declarations in the document).

Contexts are often verbatim extracts from an HTML document. Depending on how you wish to use the contexts, you may need to remove extraneous newline, tab, and whitespace characters. If you wish to present the results in  HTML or XML, you should [escape](http://stackoverflow.com/questions/7381974/which-characters-need-to-be-escaped-on-html) the contexts.

Here's a simple example of how a List of `Assertion` can be displayed to a user:
```java
/* Runs the i18n-checker on the given URL and prints the results to the
 * standard output stream. */
private static void printI18nCheck(java.net.URL url)
        throws java.io.IOException {
    // Run the checker on the remote Web page.
    java.util.List<org.w3.i18n.Assertion> results =
            org.w3.i18n.I18nChecker.check(url);
    // Print the results.
    for (Assertion assertion : results) {
        System.out.printf(
                "(%s) %s\n    Context: %s\n",
                assertion.getLevel(),
                assertion.getHtmlTitle(),
                assertion.getContexts()
                // Remove unwanted whitespace from verbatim contexts.
                .toString().replaceAll("\\s+", " "));
    }
}
```

The output generated by this method might look like:
```
…
(INFO) Charset encoding declared in 'Content-Type' HTTP response header
    Context: [utf-8, Content-Type: text/html; charset=utf-8]
(INFO) Character encoding attribute in a <code>meta</code> tag
    Context: [utf-8, <meta charset="utf-8">]
(INFO) Document Type Definition (DOCTYPE)
    Context: [HTML5, <!DOCTYPE html>]
(INFO) Language attribute in an opening <code>html</code> tag
    Context: [en, <html lang="en" >]
(ERROR) <code>charset</code> attribute used on <code>link</code> or <code >a</code> elements
    Context: [<link charset="utf-8" href="main.css" >]
…
```

Developers
----------
**Please note:**  This project is still in the early stages of its development.

The **i18n-checker** has a simple design. The two classes,  [`org.w3.i18n.I18nChecker`](http://github.com/w3c/i18n-checker/blob/master/src/main/java/org/w3/i18n/I18nChecker.java) and [`org.w3.i18n.Assertion`](http://github.com/w3c/i18n-checker/blob/master/src/main/java/org/w3/i18n/Assertion.java), form the checker's API (as described above). Then there's:
* [`org.w3.i18n.DocumentResource`](http://github.com/w3c/i18n-checker/blob/master/src/main/java/org/w3/i18n/DocumentResource.java), which represents a remote or local document to be checked;
* [`org.w3.i18n.ParsedDocument`](http://github.com/w3c/i18n-checker/blob/master/src/main/java/org/w3/i18n/ParsedDocument.java), which is constructed with a `DocumentResource`, and is responsible for processing and analysing the document; and
* [`org.w3.i18n.Check`](http://github.com/w3c/i18n-checker/blob/master/src/main/java/org/w3/i18n/Check.java), which uses the information prepared by `ParsedDocument` to create a list of assertions-- i.e. the results!

The relationship between `ParsedDocument` and `Check` is similar to the relationship between the _model_ and the _view_ in the Model–view–controller architectural pattern: The `ParsedDocument` contains most of the business logic, and does all the hard work of analysing a document, then the `Check` takes this information and renders it in to a list of `Assertion`.

###To extend the functionality of the checker

First, extend or expand `ParsedDocument`. Instances of `ParsedDocument` already have access to the document body, a map of HTTP response headers, a document model from the HTML parser (currently a `Source` object), and other details. Add your extra logic for processing the document, and then expose something useful for a `Check` object to use.

Second, extend or expand `Check`. Glance at `Check` and you'll notice: (1) a `Check` is instantiated with a `ParsedDocument`; (2) in the constructor, a list of `Assertion` is constructed; and (3) several methods that look like `private void addAssertionXYZ()` are called, which each add an `Assertion` to the list. (One of these methods might very well add nothing to the list if it finds nothing to report.) Add a new `addAssertionXYZ()` method, and have it use the extra details you exposed in `ParsedDocument`-- try to extract some really useful contexts for the end user. (NB: Make sure the new method gets called when `Check` is instantiated.)

Thirdly, your new `addAssertionXYZ()` method must construct a new `Assertion`. An `Assertion` has an ID, level, HTML title, HTML description, and a list of contexts. You can construct the assertion directly:
```java
new Assertion(
        "example_id",
        Assertion.Level.MESSAGE,
        "This is an example",
        "You needn't take any action relating to this message.",
        Arrays.asList("example context1", "example context2"));
```

Or you can make use of the [`org.w3.i18n.AssertionProvider`](http://github.com/w3c/i18n-checker/blob/master/src/main/java/org/w3/i18n/AssertionProvider.java) class. `AssertionProvider` exposes `public static Assertion getForWith(String id, Assertion.Level level, List<String> contexts)`. This method takes the id and Level you provide, and finds a corresponding title and description. Then it creates a new Assertion with the details, plus the given list of contexts. `AssertionProvider` currently gets its definitions from a properties file: [`src/main/resources/assertions-EN.properties`](https://github.com/w3c/i18n-checker/blob/master/src/main/resources/assertions-EN.properties). To add your own definitions to `AssertionProvider`, you must modify this file (instructions are provided at the top of the file in a comment block).

The following guidelines should be observed:
* `ParsedDocument` shouldn't make use of `Assertion`;
* `Check` shouldn't make use of `DocumentResource`;
* methods in `Check` shouldn't need to do any complicated processing, such as using the HTML parser; and
* assertions are written for people who want to improve the i18n of their work-- advice is qualified by published W3C guidelines and specifications.

Testing
-------
The **i18n-checker** uses the JUnit testing framework. Test fixtures for various classes can be found in [`src/test/java/org/w3/i18n/`](http://github.com/w3c/i18n-checker/blob/master/src/test/java/org/w3/i18n/). Developers should expand these fixtures as they further develop the checker.

In addition to the regular unit tests, the classes [`org.w3.i18n.I18nTest`](http://github.com/w3c/i18n-checker/blob/master/src/test/java/org/w3/i18n/I18nTest.java) and [`org.w3.i18n.I18nTestRunnerTest`](http://github.com/w3c/i18n-checker/blob/master/src/test/java/org/w3/i18n/I18nTestRunnerTest.java) make use of legacy testing resources created for the old checker [old i18n checker](http://validator.w3.org/i18n-checker/). Have a look at these classes and the properties files in [`src/test/resources/`](http://github.com/w3c/i18n-checker/blob/master/src/test/resources/) to get a feel for how the legacy resources are used.

License
-------
The **i18n-checker** is released under the [W3C® Software License](http://www.w3.org/Consortium/Legal/2002/copyright-software-20021231).

> i18n-checker: https://github.com/w3c/i18n-checker
>
> Copyright © 2013 World Wide Web Consortium, (Massachusetts Institute of Technology, European Research Consortium for Informatics and Mathematics, Keio University, Beihang). All Rights Reserved. This work is distributed under the W3C® Software License [1] in the hope that it will be useful, but WITHOUT ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
> 
> [1] http://www.w3.org/Consortium/Legal/2002/copyright-software-20021231
