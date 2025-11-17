## What is an Index?

https://www.youtube.com/watch?v=aZjYr87r1b8

```
At its core, a database index is pretty much like an index in a book - it allows you to quickly find information without having to read through all of the content. In a database, indexing involves creating a structured roadmap that speeds up the retrieval of specific information from a huge collection of data.

Imagine you’ve created a movie database app. Users want to explore films based on filters: genre, release year, lead actors, and more. But as your user base grows, running a search for these filters becomes as slow as watching a buffering video.

Here's where indexes step in, like movie genres neatly organized on your shelf. Let's say you decide to index the "release year" column. Behind the scenes, your database engine creates a digital list with two columns: a release year value and a link to the corresponding movie record.

When a user hunts for movies from a specific year, your query instantly knows where to find the desired films. Thanks to this index magic, the query's efficiency is supercharged, taking mere seconds instead of super long wait times.

Without indexing, databases might need to scan through every row of data to find the desired information. By creating a roadmap that guides database systems to relevant data points, indexing can reduce the time required for queries and can help ensure that query tasks are performed quickly, without unnecessary strain on system resources.
```
```
A database index is a specialized data structure created on one or more columns of a table that allows the database engine to find rows more quickly without scanning the entire table. Think of it as a sorted list of keys from the indexed column(s) along with pointers (references) to the actual data rows. The main goal is to speed up query filtering and sorting operations. For example, an index on a "company_id" column would hold the sorted company_id values plus pointers to rows where these IDs exist, drastically reducing search time
```

## resources

https://www.atlassian.com/data/databases/how-does-indexing-work

https://xata.io/blog/xatabyte-database-indexing