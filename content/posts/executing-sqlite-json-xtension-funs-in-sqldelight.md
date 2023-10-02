+++
title = '''Running SQLite's JSON Functions in SQLDelight'''
date = 2023-10-02T19:37:51Z
draft = true
categories = ["KMP"]
tags = ["kmp", "sqlite", "sqldelight", "json_group_array"]
+++

I have been working on a side project using [Kotlin Multiplatform](https://kotlinlang.org/lp/multiplatform/) and [SQLDelight](https://cashapp.github.io/sqldelight/2.0.0/). Today, I had to write the query below: 

```sql
-- Queries.sq

getWordAndDescriptions:
SELECT table1.id, table1.word, table2.description
FROM table1 
LEFT JOIN table2
ON table1.word = table2.word
GROUP BY table1.word
```

Here's an example result set from the query above: 
<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: center;">
      <th>Word</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Season</td>
      <td>1st description of season</td>
    </tr>
        <tr>
      <td>Season</td>
      <td>2nd description of season</td>
    </tr>
        <tr>
      <td>Season</td>
      <td>3rd description of season</td>
    </tr>
        <tr>
      <td>Victory</td>
      <td>1st description of victory</td>
    </tr>
        <tr>
      <td>Victory</td>
      <td>2nd description of season</td>
    </tr>
        <tr>
      <td>Winner</td>
      <td>1st description of winner</td>
    </tr>
  </tbody>
</table>
</div>

You would notice that Season and Victory are repeating in the `word` column. Typically I would write some code in my repository to generate a key-value mapping from this result set. This mapping would've been of type `Map<String, List<String>>` or something similar. Fortunately, I kept asking myself if it was necessary for me to do all that by hand? So I dived into the SQLite docs and discovered `json_group_array(expr)`. 

From the docs
>`JSON_GROUP_ARRAY(expr)` is an aggregate function that returns a JSON array comprised of all values in the aggregation. 

This is exactly what I need! I could use it to aggregate the results in the `description` column. The query can be rewritten to look like this: 
``` sql
-- Queries.sq

getWordWithMeanings:
SELECT table1.id, table1.word,
json_group_array(table2.description) AS meaning
FROM table1 
LEFT JOIN table2
ON table1.word = table2.word
GROUP BY table1.word
```

<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: center;">
      <th>Word</th>
      <th>Meaning</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Season</td>
      <td>[1st description of season, 2nd desc..., 3rd desc]</td>
    </tr>
        <tr>
      <td>Victory</td>
      <td>[1st desc. of victory, 2nd desc of victory]</td>
    </tr>
    <tr>
      <td>Winner</td>
      <td>[1st description of winner]</td>
    </tr>
  </tbody>
</table>
</div>

The table above is the result set from our new query. Perfect! 
## SQLDelight
Wait. There is a compile-time error. SQLDelight doesn't know the `json_group_array` function. Huh?

After some digging(the SQLDelight docs aren't clear on this) I realized I had to add the `sqlite-json-module` in the multiplatform `build.gradle.kts` file that had the `sqldelight {}` block. Doing this added support for the JSON extensions in SQLite to the project, thereby making `json_group_array` visible to the compiler: 
``` groovy
// build.gradle.kts

sqldelight {  
    databases {  
        create("dbName") {  
            packageName.set("com.package.name")  
            // line to add
            module("app.cash.sqldelight:sqlite-json-module:${VersionNumber}")   
        }  
    }
}
```
Sync the project and the compiler will stop complaining. 

As we know, SQLDelight "generates type-safe Kotlin APIs from our SQL statements". This means it's going to generate the model below from our query: 
``` kotlin
// Generated GetWordWithMeaning.kt

data class GetWordWithMeaning(  
	val id: Long,  
	val text: String,  
	val meaning: String,  
)
```
Notice that SQLDelight did not generate a `List<String>` type for our `meaning` property, rather the data type of `meaning` is a `String` to represent the JSON String, which is the result of `json_group_array`. We can deserialize this using a JSON (de)serialization library of choice. 
Using the [Kotlin Serialization](https://github.com/Kotlin/kotlinx.serialization) library: 
```kotlin
// SharedWordRepository.kt

fun getWordsWithTheirMeanings(): Flow<List<WordAndMeanings>> { 
	return database.queries
		.getWordWithMeanings(mapper = { id, word, meaningsJson -> // 1
			WordAndMeanings(
				id = id, 
				word = word, 
				meanings = Json.decodeFromString(meaningsJson) // 2
			)
		}
	)
}


data class WordAndMeanings( 
	val id: Long, 
	val word: String, 
	val meanings: List<String> = emptyList()
)
```

From the code snippet above: 
1. We pass a lambda function with 3 arguments (matches the columns in the query) in order to transform the results from the generated model into a `WordAndMeanings` model 
2. `meanings = Json.decodeFromString(meaningsJson)` : On this line we deserialize the resulting JSON string into a `List<String>`. 

And everything works as it should with minimum fuss! Hopefully this helps someone out there, who knows? 