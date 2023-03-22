---
title: Searching within JSON fields in Postgres, with EFCore 3
draft: false
date: 2021-11-24T15:15:33Z
author: PolarizedIons
---

With it sounding so simple, you wouldn't expect the hoops you'd have to go though!

## Initial setup

My use-case is translations. I don't want to join on another table every time, and want to be able to scale to add new languages easily. To do that, I'm going to make a strongly typed class called `Translations` that extends `Dictionary<string, Dictionary<string, string>>`. The outer key is the language, and the inner key is the translation key. The value is the translated string. That's all I really need, but for the sake of this blog post, I'll add a helper method to add translations easily, making our class look like this:

```csharp
public class Translations : Dictionary<string, Dictionary<string, string>>
{
    public Translations Add(string language, string key, string value)
    {
        if (!ContainsKey(language))
        {
        	this[language] = new Dictionary<string, string>();
    	}

    	this[language][key] = value;
	    return this;
	}
}
```

My entity I'll be using to test this with, is very simple, just an id and the translations, making sure to specify to EFCore that it's of `json` type.

```csharp
public class MyEntity
{
	[Key]
	public Guid Id { get; set; }

	[Column(TypeName = "json")]
	public Translations Translations { get; set; }
}
```

I'll create an initial migration, and add a few entires:

{{< figure src="img/image_o-43.png" >}}

Then print out what we have:

{{< figure src="img/image_o-44.png" >}}

{{< figure src="img/image_o-42.png" >}}

## The problem

Now, _selecting_ the German strings works, so what's the problem?

{{< figure src="img/image_o-46.png" >}}

{{< figure src="img/image_o-48.png" >}}

It's when we _search_ for them!

{{< figure src="img/image_o-47.png" >}}

{{< figure src="img/image_o-49.png" >}}

We get a very nice exception, saying it can't translate our linq to SQL. So how do we solve this?

## EFCore DB Function mappings!

While [the documentation](https://docs.microsoft.com/en-us/ef/core/querying/user-defined-function-mapping) is a bit lacking, it's a very simple concept: We create a strong typed method in our code, and tell EFCore _"when you see this method, call this SQL instead"_.

First, we need to create that strong typed method.

```csharp
public static class MyDbFunctions
    {
        public static string GetTranslations(Translations column, string language, string key) =>
            throw new NotImplementedException("Must be called on the db!");
    }
```

EFCore will replace its functionality when translating to SQL, so it doesn't matter what we make it's value. For simplicity's sake, I just throw an exception. This will only get called when it does _client side evaluation_.

Next, we'll tell EFCore about it, in the `OnModelCreating` method in our DatabaseContext:

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            var getTranslationsFunction = modelBuilder
                .HasDbFunction(typeof(MyDbFunctions).GetMethod(nameof(MyDbFunctions.GetTranslations)))
                .HasTranslation(args =>
                    SqlFunctionExpression.Create("json_extract_path_text", args, typeof(string), new StringTypeMapping("NVARCHAR(MAX)"))
                );
            getTranslationsFunction.HasParameter("column").Metadata.TypeMapping = new StringTypeMapping("NVARCHAR(MAX)");
            getTranslationsFunction.HasParameter("language").Metadata.TypeMapping = new StringTypeMapping("NVARCHAR(MAX)");
            getTranslationsFunction.HasParameter("key").Metadata.TypeMapping = new StringTypeMapping("NVARCHAR(MAX)");

            base.OnModelCreating(modelBuilder);
        }
```

Here, we just map the `MyDbFunctions.GetTranslations` method, to a postgres method, `[json_extract_path_text](https://www.postgresql.org/docs/current/functions-json.html)`, and tell EFCore to expect 3 string arguments.

You are obviously not just limited to that, you can do any SQL you want. Some of the more advanced  Postgresql functions can be found [here](https://www.postgresql.org/docs/current/functions-json.html).

_A side note: json fields always return a **string**. Even if you do something with `json_array_elements`, which splits an array into its elements, the result will be **json strings** -  so don't try to make the return type a `string[]`, it won't work! EFCore will be smart enough to return multiple rows, so also no need for a `.SelectMany` in your query._

Now, we can simple do:

```csharp
var entities = _db.MyEntities
                .Where(x => 
                    MyDbFunctions.GetTranslations(x.Translations, "en", "title").Contains("new")
                );
```

... and when we print it out, we only get one of the two entities we created:

{{< figure src="img/image_o-50.png" >}}

The same for if we search for German text:

{{< figure src="img/image_o-51.png" >}}

And if I turn back on the debug logs for executing sql, we can see it translated it correctly _on the database_!

```sql
SELECT m."Id", m."Translations"
FROM "MyEntities" AS m
WHERE STRPOS(json_extract_path_text(m."Translations", 'de', 'title'), 'Meine') > 0

```

## Wrapping up

This is was a headache and a half for me to figure out, so hopefully it can be of use to someone else!

Code for this post can be [found on GitHub](https://github.com/PolarizedIons/EFCoreExperimentJsonSearching).

