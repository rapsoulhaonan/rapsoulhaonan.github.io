---
title: "How to: Bind to XML Data Using an XMLDataProvider and XPath Queries"
header-img: "img/post-bg-data-bind.jpg"
tags:
  - C#
  - XML
---

This example shows how to bind to XML data using an [XmlDataProvider](https://docs.microsoft.com/en-us/dotnet/api/system.windows.data.xmldataprovider?view=netframework-4.7.2).  
  
 With an XmlDataProvider, the underlying data that can be accessed through data binding in your application can be any tree of XML nodes. In other words, an XmlDataProvider provides a convenient way to use any tree of XML nodes as a binding source.

**Example**

In the following example, the data is embedded directly as an XML data island within the Resources section. An XML data island must be wrapped in <x:XData> tags and always have a single root node, which is Inventory in this example.

> Note

> The root node of the XML data has an xmlns attribute that sets the XML namespace to an empty string. This is a requirement for applying XPath queries to a data island that is inline within the XAML page. In this inline case, the XAML, and thus the data island, inherits the System.Windows namespace. Because of this, you need to set the namespace blank to keep XPath queries from being qualified by the System.Windows namespace, which would misdirect the queries.

XAML
{% highlight XML %}

<StackPanel
  xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
  xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
  Background="Cornsilk">

  <StackPanel.Resources>
    <XmlDataProvider x:Key="InventoryData" XPath="Inventory/Books">
      <x:XData>
        <Inventory xmlns="">
          <Books>
            <Book ISBN="0-7356-0562-9" Stock="in" Number="9">
              <Title>XML in Action</Title>
              <Summary>XML Web Technology</Summary>
            </Book>
            <Book ISBN="0-7356-1370-2" Stock="in" Number="8">
              <Title>Programming Microsoft Windows With C#</Title>
              <Summary>C# Programming using the .NET Framework</Summary>
            </Book>
            <Book ISBN="0-7356-1288-9" Stock="out" Number="7">
              <Title>Inside C#</Title>
              <Summary>C# Language Programming</Summary>
            </Book>
            <Book ISBN="0-7356-1377-X" Stock="in" Number="5">
              <Title>Introducing Microsoft .NET</Title>
              <Summary>Overview of .NET Technology</Summary>
            </Book>
            <Book ISBN="0-7356-1448-2" Stock="out" Number="4">
              <Title>Microsoft C# Language Specifications</Title>
              <Summary>The C# language definition</Summary>
            </Book>
          </Books>
          <CDs>
            <CD Stock="in" Number="3">
              <Title>Classical Collection</Title>
              <Summary>Classical Music</Summary>
            </CD>
            <CD Stock="out" Number="9">
              <Title>Jazz Collection</Title>
              <Summary>Jazz Music</Summary>
            </CD>
          </CDs>
        </Inventory>
      </x:XData>
    </XmlDataProvider>
  </StackPanel.Resources>

  <TextBlock FontSize="18" FontWeight="Bold" Margin="10"
    HorizontalAlignment="Center">XML Data Source Sample</TextBlock>
  <ListBox
    Width="400" Height="300" Background="Honeydew">
    <ListBox.ItemsSource>
      <Binding Source="{StaticResource InventoryData}"
               XPath="*[@Stock='out'] | *[@Number>=8 or @Number=3]"/>
    </ListBox.ItemsSource>

    <!--Alternatively, you can do the following. -->
    <!--<ListBox Width="400" Height="300" Background="Honeydew"
      ItemsSource="{Binding Source={StaticResource InventoryData},
      XPath=*[@Stock\=\'out\'] | *[@Number>\=8 or @Number\=3]}">-->

    <ListBox.ItemTemplate>
      <DataTemplate>
        <TextBlock FontSize="12" Foreground="Red">
          <TextBlock.Text>
            <Binding XPath="Title"/>
          </TextBlock.Text>
        </TextBlock>
      </DataTemplate>
    </ListBox.ItemTemplate>
  </ListBox>
</StackPanel>
{% endhighlight %}

As shown in this example, to create the same binding declaration in attribute syntax you must escape the special characters properly. For more information, see [XML Character Entities and XAML](https://docs.microsoft.com/en-us/dotnet/framework/xaml-services/xml-character-entities-and-xaml).

The ListBox will show the following items when this example is run. These are the Titles of all of the elements under Books with either a Stock value of "out" or a Number value of 3 or greater than or equals to 8. Notice that no CD items are returned because the [XPath](https://docs.microsoft.com/en-us/dotnet/api/system.windows.data.xmldataprovider.xpath?view=netframework-4.7.2) value set on the XmlDataProvider indicates that only the Books elements should be exposed (essentially setting a filter).

In this example, the book titles are displayed because the XPath of the [TextBlock](https://docs.microsoft.com/en-us/dotnet/api/system.windows.controls.textblock?view=netframework-4.7.2) binding in the [DataTemplate](https://docs.microsoft.com/en-us/dotnet/api/system.windows.datatemplate?view=netframework-4.7.2) is set to "Title". If you want to display the value of an attribute, such as the ISBN, you would set that XPath value to "@ISBN".

The XPath properties in WPF are handled by the XmlNode.SelectNodes method. You can modify the XPath queries to get different results. Here are some examples for the XPath query on the bound [ListBox](https://docs.microsoft.com/en-us/dotnet/api/system.windows.controls.listbox?view=netframework-4.7.2) from the previous example:

1. **XPath="Book[1]"** will return the first book element ("XML in Action"). Note that XPath indexes are based on 1, not 0.

2. **XPath="Book[@*]"** will return all book elements with any attributes.

3. **XPath="Book[last()-1]"** will return the second to last book element ("Introducing Microsoft .NET").

4. **XPath="*[position()>3]"** will return all of the book elements except for the first 3.

When you run an XPath query, it returns an XmlNode or a list of XmlNodes. XmlNode is a common language runtime (CLR) object, which means you can use the Path property to bind to the common language runtime (CLR) properties. Consider the previous example again. If the rest of the example stays the same and you change the TextBlock binding to the following, you will see the names of the returned XmlNodes in the ListBox. In this case, the name of all the returned nodes is "Book".

XAML
{% highlight XML %}

<TextBlock FontSize="12" Foreground="Red">
  <TextBlock.Text>
    <Binding Path="Name"/>
  </TextBlock.Text>
</TextBlock>
In some applications, embedding the XML as a data island within the source of the XAML page can be inconvenient because the exact content of the data must be known at compile time. Therefore, obtaining the data from an external XML file is also supported, as in the following example:
{% endhighlight %}

XAML
{% highlight XML %}

<XmlDataProvider x:Key="BookData" Source="data\bookdata.xml" XPath="Books"/>
If the XML data resides in a remote XML file, you would define access to the data by assigning an appropriate URL to the [Source](https://docs.microsoft.com/en-us/dotnet/api/system.windows.data.xmldataprovider.source?view=netframework-4.7.2) attribute as follows:
{% endhighlight %}

XML
{% highlight XML %}

<XmlDataProvider x:Key="BookData" Source="http://MyUrl" XPath="Books"/>  
{% endhighlight %}

**See Also**

[ObjectDataProvider](https://docs.microsoft.com/en-us/dotnet/api/system.windows.data.objectdataprovider?view=netframework-4.7.2)

[Bind to XDocument, XElement, or LINQ for XML Query Results](https://docs.microsoft.com/en-us/dotnet/framework/wpf/data/how-to-bind-to-xdocument-xelement-or-linq-for-xml-query-results)

[Use the Master-Detail Pattern with Hierarchical XML Data](https://docs.microsoft.com/en-us/dotnet/framework/wpf/data/how-to-use-the-master-detail-pattern-with-hierarchical-xml-data)

[Binding Sources Overview](https://docs.microsoft.com/en-us/dotnet/framework/wpf/data/binding-sources-overview)

[Data Binding Overview](https://docs.microsoft.com/en-us/dotnet/framework/wpf/data/data-binding-overview)

[How-to Topics](https://docs.microsoft.com/en-us/dotnet/framework/wpf/data/data-binding-how-to-topics)




{% video https://github.com/rapsoulhaonan/rapsoulhaonan.github.io/blob/master/video/Harry-Potter.mp4 240 480 https://github.com/rapsoulhaonan/rapsoulhaonan.github.io/blob/master/img/404-bg.jpg %}

<iframe width="240" height="480" src="https://github.com/rapsoulhaonan/rapsoulhaonan.github.io/blob/master/video/Harry-Potter.mp4" frameborder="0" allowfullscreen></iframe>

