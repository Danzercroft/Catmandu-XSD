# NAME

Catmandu::XSD - Modules for handling XML data with XSD compilation

# SYNOPSIS

    ## Converting XML to YAML/JSON/CSV/etc

    # Compile an XSD schema file and parse one shiporder.xml file
    catmandu convert XSD --root '{}shiporder'
                         --schemas demo/order/*.xsd
                         to YAML < shiporder.xml

    # Same as above but parse more than one file into an array of records
    catmandu convert XSD --root '{}shiporder'
                         --schemas demo/order/*.xsd
                         --files 'data/*.xml'
                         to YAML

    # Same as above but all array of records are in a XML container file
    catmandu convert XSD --root '{}shiporder'
                         --schemas demo/order/*.xsd
                         --xpath '/Container/List//Record/Payload/*'
                         to YAML < data/container.xml

    ## Convert an YAML/JSON/CSV into XML validated against an XSD schemas

    # Convert one shiporder YAML to XML
    catmandu convert YAML to XSD --root '{}shiporder'
                                 --schemas demo/order/*.xsd < shiporder.YAML

    # Same as above but store multiple shiporders in the YAML into a separate file
    catmandu convert YAML to XSD --root '{}shiporder'
                                 --schemas demo/order/*.xsd
                                 --split 1
                                 < shiporder.YAML

    # Same as above but use template toolkit to pack the XML into an container
    # (The xml record is stored in the 'xml' key which can be retrieved in the
    # template by [% xml %])
    catmandu convert YAML to XSD --root '{}shiporder'
                                 --schemas demo/order/*.xsd
                                 --template_before t/xml_header.tt
                                 --template t/xml_record.tt
                                 --template t/xml_footer.tt
                                 < shiporder.YAML

     ## Example documents

     # Show an example how a valid XML document needs to be structured for an
     # XSD scheme.
     catmandu convert XSD --root {}shiporder
                          --schemas "t/demo/order/*xsd"
                          --example 1 to YAML

# DESCRIPTION

[Catmandu::XSD](https://metacpan.org/pod/Catmandu::XSD) contains modules for handling XML data within the [Catmandu](https://metacpan.org/pod/Catmandu)
framework. Parsing and serializing is based on [XML::Compile](https://metacpan.org/pod/XML::Compile).

There are two modules available for handling XML data in the Catmandu framework:
[Catmandu::XML](https://metacpan.org/pod/Catmandu::XML) and [Catmandu::XSD](https://metacpan.org/pod/Catmandu::XSD). The former one can be used when no XML schema
is available for the data. It provides a simple interface to read in XML data and
transform it to other formats. Because [Catmandu::XML](https://metacpan.org/pod/Catmandu::XML) doesn't depend on an
XSD schema, it can't know which fields in the input XML files are sequences or
single value elements. Each record is parsed on its own. A record with content:

    <foo>
      <bar>test</bar>
    </foo>

will be parsed into a YAML output like:

    catmandu XML to YAML < test.xml
    --
    bar: test

A record with content:

    <foo>
      <bar>test</bar>
      <bar>test</bar>
    </foo>

will be parsed into a YAL output like:

    catmandu XML to YAML < test2.xml
    --
    bar:
      - test
      - test

In the first case 'bar' will contain a string, in the second case an array. This
might no be what you want in some programming projects. E.g. when you need the 'bar'
field to be always an array of values, then you an XSD schema file is required
containing the exact structure of the XML document:

    test.xsd:
    <?xml version="1.0" encoding="UTF-8" ?>
    <xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema">
        <xs:element name="foo">
         <xs:complexType>
          <xs:sequence>
            <xs:element name="bar" type="xs:string" maxOccurs="unbounded"/>
          </xs:sequence>
         </xs:complexType>
        </xs:element>
    </xs:schema>

And now the test.xml and test2.xml can be parsed with help of Catmandu::XSD:

    catmandu XSD --root '{}foo' --schemas test.xsd to YAML < test.xml
    --
    bar:
      - test

    catmandu XSD --root '{}foo' --schemas test.xsd to YAML < test2.xml
    --
    bar:
      - test
      - test

# WILDCARDS

Some XSD Schema allow for `any` or `anyAttribute` specifications in the schema.
The [Catmandu::XSD](https://metacpan.org/pod/Catmandu::XSD) modules can't guess in these cases what the schema implementation
is. These nodes will be parsed as [XML::LibXML::Node](https://metacpan.org/pod/XML::LibXML::Node)s in the
resulting documents. Catmandu output formats such as [Catmandu::Exporter::JSON](https://metacpan.org/pod/Catmandu::Exporter::JSON)
can't handle these XML::LibXML::Node nodes. You have to implement yourself a
[Catmandu::Fix](https://metacpan.org/pod/Catmandu::Fix) to translate these values in to plain string, array or hash elements.

But in general a round trip should be problematic:

    catmandu XSD --root ... --schema wildcard.xsd to XSD  --root ... --schema wildcard.xsd < data.xml

# MIXED ELEMENTS

ComplexType and ComplexContent in the XSD schema can be declared with the `<mixed="true"`> attribute.
This means that in the XML documents simple text and XML elements can be mixed as in:

      Hello, I'm <name>John</name> how can I <bold>help</bold> you?

In these cases it is not know if the elements are required as an hash or should be ignored. By
defaults [Catmandu::XSD](https://metacpan.org/pod/Catmandu::XSD) will parse these elements as [XML::LibXML::Node](https://metacpan.org/pod/XML::LibXML::Node)s documents.
This behavious can be changed by setting the 'mixed' flag:

    # All mixed elements will be XML::LibXML::Node-s
    catmandu XSD --root ... --schema mixed.xsd  < data.xml

    # The mixed elements will be ignored, only the text will survive
    #
    #  Hello, I'm <name>John</name> how can I <bold>help</bold> you?
    #
    #  =>  Hello, I'm John how can I help you?
    catmandu XSD --root ... --schema mixed.xsd --mixed TEXTUAL < data.xml

    # The mixed text will be ignored, only the elements will survive
    #
    #  Hello, I'm <name>John</name> how can I <bold>help</bold> you?
    #
    #  =>  { name => 'John' , bold => 'help' }
    catmandu XSD --root ... --schema mixed.xsd --mixed STRUCTURAL < data.xml

    # The mixed elements will be a plain XML fragment string
    #
    #  Hello, I'm <name>John</name> how can I <bold>help</bold> you?
    #
    #  =>  $r = 'Hello, I'm <name>John</name> how can I <bold>help</bold> you?'
    catmandu XSD --root ... --schema mixed.xsd --mixed XML_STRING < data.xml

# MODULES

- [Catmandu::Importer::XSD](https://metacpan.org/pod/Catmandu::Importer::XSD)

    Parse and validate XML data using an XSD file for structural data.

- [Catmandu::Exporter::XSD](https://metacpan.org/pod/Catmandu::Exporter::XSD)

    Serialize and validate XML data using an XSD file for structural data.

- [Catmandu::Fix::xpath\_map](https://metacpan.org/pod/Catmandu::Fix::xpath_map)

    Map XML from XSD-any elements into data fields using XPath expressions.

# BUGS, QUESTIONS, HELP

Use the github issue tracker for any bug reports or questions on this module:
https://github.com/LibreCat/Catmandu-XSD/issues

# DISCLAIMER

This module is based on [XML::Compile](https://metacpan.org/pod/XML::Compile) and the [Catmandu](https://metacpan.org/pod/Catmandu) framework.

[XML::Compile](https://metacpan.org/pod/XML::Compile) is the workhorse that forms the core of this module to
compile XSD file into parser and serializers.

[Catmandu](https://metacpan.org/pod/Catmandu) is used to transform parsed XML into any format you like.
Catmandu contains a simple DSL languages called [Catmandu::Fix](https://metacpan.org/pod/Catmandu::Fix) to create
small scripts to manipulate data. The [Catmandu](https://metacpan.org/pod/Catmandu) toolkit is used by many
university libraries to process metadata collections.

For more information on Catmandu visit: http://librecat.org/Catmandu/
or follow the blog posts at: https://librecatproject.wordpress.com/

# AUTHOR

Patrick Hochstenbach , `patrick.hochstenbach at ugent.be`

# LICENSE AND COPYRIGHT

This program is free software; you can redistribute it and/or modify it
under the terms of either: the GNU General Public License as published
by the Free Software Foundation; or the Artistic License.

See [http://dev.perl.org/licenses/](http://dev.perl.org/licenses/) for more information.

# SEE ALSO

[XML::Compile](https://metacpan.org/pod/XML::Compile) , [Catmandu](https://metacpan.org/pod/Catmandu) , [Template](https://metacpan.org/pod/Template) , [Catmandu::XML](https://metacpan.org/pod/Catmandu::XML)
