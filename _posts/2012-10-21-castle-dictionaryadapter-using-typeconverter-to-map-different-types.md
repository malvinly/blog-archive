---
layout: post
title: "Castle DictionaryAdapter - Using TypeConverter to map different types"
date: 2012-10-21 04:04:03 +0000
categories: ["General"]
original_url: https://nivlam.wordpress.com/2012/10/20/castle-dictionaryadapter-using-typeconverter-to-map-different-types/
---

I've posted about the `DictionaryAdapter` a few times in the past ([here](http://malvinly.com/2011/03/03/strongly-typed-asp-net-session/) and [here](http://malvinly.com/2011/02/28/strongly-typed-dictionaries/)). There are scenarios when we want to map more than just simple types to strings. For example, we have an interface `IPerson` with the property `Birthday` representing the date in ticks.

{% raw %}
```csharp
public interface IPerson
{
    string Name { get; set; }
    int Age { get; set; }
    long Birthday { get; set; }
}
```
{% endraw %}

If we attempt to map a `DateTime` to the property `Birthday`, an `InvalidCastException` will be thrown when we access the `Birthday` property.

{% raw %}
```csharp
var dictionary = new Dictionary<string, object>
{ 
    { "Name", "John Doe" },
    { "Age", 30 },
    { "Birthday", DateTime.Now },
};

IPerson person = new DictionaryAdapterFactory()
	.GetAdapter<IPerson>(dictionary);

Console.WriteLine(person.Birthday);
```
{% endraw %}

Fortunately, the `DictionaryAdapter` allows us to use attributes to define how properties should be mapped by creating a class that inherits from `System.ComponentModel.TypeConverter`.

{% raw %}
```csharp
public class DateTimeToTicks : TypeConverter
{
	public override bool CanConvertFrom(ITypeDescriptorContext context, Type sourceType)
	{
		if (sourceType == typeof(DateTime))
			return true;
		
		return base.CanConvertFrom(context, sourceType);
	}

	public override object ConvertFrom(ITypeDescriptorContext context, CultureInfo culture, object value)
	{
		return ((DateTime)value).Ticks;
	}
}
```
{% endraw %}

We can decorate the `Birthday` property with the `TypeConverter` attribute and specify that we want to use our new type converter.

{% raw %}
```csharp
[TypeConverter(typeof(DateTimeToTicks))]
long Birthday { get; set; }
```
{% endraw %}

The previous example is a bit contrived. However, I have run into issues when trying to use the `DictionaryAdapter` with a result set from a database that contains `NULL`s. Here we've updated the `IPerson` interface with a `Nullable<int>`.

{% raw %}
```csharp
public interface  IPerson
{
    string Name { get; set; }
    int? Age { get; set; }
}
```
{% endraw %}

The following code will throw an `InvalidCastException` on line 17 after mapping the results from a SQL query:

{% raw %}
```csharp
using (var conn = new SqlConnection(connStr))
using (var cmd = new SqlCommand("select 'John Doe' as [Name], NULL as [Age]", conn))
{
    conn.Open();

    using (SqlDataReader dr = cmd.ExecuteReader())
    {
        while (dr.Read())
        {
            var dictionary = Enumerable
                .Range(0, dr.FieldCount)
                .ToDictionary(dr.GetName, dr.GetValue);

            IPerson person = new DictionaryAdapterFactory()
                .GetAdapter<IPerson>(dictionary);

            Console.WriteLine(person.Age);
        }
    }
}    
```
{% endraw %}

We can create another `TypeConverter` to check for `DBNull` and returns `null` if found.

{% raw %}
```csharp
public class DBNullable : TypeConverter
{
    public override bool CanConvertFrom(ITypeDescriptorContext context, Type sourceType)
    {
        if (sourceType == DBNull.Value.GetType())
            return true;

        return base.CanConvertFrom(context, sourceType);
    }

    public override object ConvertFrom(ITypeDescriptorContext context, CultureInfo culture, object value)
    {
        if (DBNull.Value.Equals(value))
            return null;

        return base.ConvertFrom(context, culture, value);
    }
}
```
{% endraw %}

Again, we can decorate the `Age` property with our new type converter.

{% raw %}
```csharp
[TypeConverter(typeof(DBNullable))]
int? Age { get; set; }
```
{% endraw %}
