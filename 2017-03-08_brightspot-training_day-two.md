# Brightspot Training: Day Two


## Reloader

The Reloader is a part of Dari, so it's also knows as the "Dari Reloader".

Here is the only existing documentation on the Reloader, fetched by Maegan Farnum:

> A prompt to install a Reloader application may appear in the top right of the browser. This is the Dari on-the-fly Java code reloader installed in the webapps directory of Tomcat. It reloads changes on-the-fly for Java code, eliminating the need to redeploy the Java application for every change. If refreshing does not prompt the reloader, add *?_reload=true* to the end of the URL or stop the project, rebuild, and redeploy.

PSD is working on some new, more robust documentation on this, of course.

`build.properties` has properties that tells where to monitor for changes to files (for example, see `brightspot-tutorial/target/tutorial-1.0-SNAPSHOT/WEB-INF/classes/build.properties`).

For example, `javaSourceDirectory` to know where the `src/main/java` dir location is, e.g. `javaSourceDirectory=/Users/162084/projects/brightspot-tutorial/src/main/java` in the `build.properties`.

When using Vagrant, the Reloader has to make sure the files on your directory is moved to in the Vagrant VM so that Vagrant also updates. The Vagrantfile maps the location on your hard drive to the locations in the Vagrant filesystem. The `/Users` directory  is symlinked in Vagrant to the hard drive's `/Users` directory.

Additionally `/etc/hosts` in Vagrant has to be mapped to the `/etc/hosts` on your hard drive. This also means you can setup an alias, e.g. modify `/etc/hosts` to include something like:

```
172.28.128.15  training.vagrant
```

Where the above IP is from the `ifconfig` from Vagrant (while ssh'd into Vagrant), and `training.vagrant` is whatever alias you want to access Vagrant from in the browser in lieu of using the IP address.

## Brightspot Modeling

Dari provides a schema for each database that never changes. The Java classes never change that schema. This also means changes never really happen to the database structure, only to the data in the existing structure - which is all in the code.

`/_debug/db-bulk`, e.g. http://training.vagrant/_debug/db-bulk

`describe Record;` results in fields, types of those fields, and other MySQL information.

You can use the Index tool of the db-bulk tool in Dari to re-index data that was saved non-indexed after an index was added for that data.

### State, Record, and Recordable

State is essentially a map of values from the database, with a corresponding ID and type ID.

Recordable is an interface that has methods for accessing a State object for a given thing in the database. Content extends Record (which implements Recordable), which allows you to do things like `this.save()` and `this.delete()` (to interface with the State).

Database.java has a bunch of methods that need to be implemented for any implementation yourself. MySQL and Solr is a typical installation. Interfaces with Postgres and Elasticsearch are being worked on.

(We should look at interfacing with the JCR).

### Many to Many Relationships

Let's say you need an Employee class and a Project class where the project has its own start and end dates, and employees can be on many projects with their own start and end dates that may not be the same as the project start and end date. This is a case for a Bridge Table in relational databases, but in the world of Dari, we are limited to Java classes.

We can create a class that has an employee, project, and dates for start and end. I named mine `EmployeeProjectTenure.java`, but in our training the name `ProjectMember.java` was chosen. That works fine, but what about if we need access to the employees and project objects both.

You can create an Interface that Project and Employee both implement. In our example, we created an interface called `DurationTrackable`.
This interface must extend `Recordable`.

### Modification

[Modifications Documentation](http://docs.brightspot.com/cms/developers-guide/content-modeling/modifications.html)

Moving start and end date into a common, shared space that is inherited and shared by the places that otherwise duplicated them, e.g. the start and end dates in Project.java and in Employee.java can be moved into a multiple inheritance pattern via interface and modification class.

ToolUserModification.java:

```
package
import com.psddev.dari.db.Modification;

public class ToolUserModification extends Modification<DurationTrackable> {

}
```

```
public class DurationTrackableData extends
```

`.as()`

```
for (DurationTrackableData dt : Query.from(DurationTrackable.class).where("startDate != missing").selectAll() {
  Duration
}
```



## Query

`Query.from(Article.class)` - the base query on which additional functions can be used, e.g. `Query.from(Article.class).selectAll()`

Query methods return different types:
```
Query<Article> query = Query.from(Article.class); // get the query
// Notice each function on the query returns something different...
List<Article> selectAll = query.selectAll(); // don't use Select All unless it's guaranteed that it's a small subset...
PaginatedResult<Article> select = query.select(0, 10);
Article first = query.first()
Iterable<Article> iterable = query.iterable(200); // gets first 200 results in memory to iterate over
for (Article article : iterable) {
  // ... do something for each of the first 200 results
}

query.fields("title"); // not sure what this does :D
```

### SQL


`/_debug/db-sql` Dari tool


```
Query<Article> query = Query.from(Article.class);
String sql = Database.Static.getFirst(SqlDatabase.class).buildSelectStatement(query); // to get the SQL for a given Dari Query object
```

### Query for Trash

```
Query<Article> query = Query.from(Article.class);
query.where("cms.content.trashed = ?", true); // will return the trashed Content
```

### Query API

[Query API reference](https://artifactory.psdops.com/psddev-releases/com/psddev/dari/3.2.2188-2d7dae/dari-3.2.2188-2d7dae-javadoc.jar!/com/psddev/dari/db/Query.html)

To be able to query data from a java model, use the `@Indexed` annotation.

To make a field essentially a primary key with the unique constraint, you can use `@Indexed(unique = true)`.

```
Query.from(Author.class)
     .where("name = ?", "Dan Slaughter");
```

#### Group By

```
Query.from(Author.class)
     .groupBy("name");
// returns something like:
// 1. 3 [dan slaughter] // meaning there are three "dan slaughter" instances in the database
// 2. 2 [hyoo lim]
// 3. 1 [jeremy collins]
```

#### Subqueries

You can make subqueries like so:

```
Query<Author> authorQuery = Query.from(Author.class)
     .where("name = ?", "Dan Slaughter");

Query.from(Article.class)
  .where("author = ?", authorQuery)
  .selectAll();
```

> Key: dari/subQueryResolveLimit Type: java.lang.Integer

> Since Solr does not currently support joins, Dari will execute subqueries separately. This limits the size of the results used to prevent generating too large of a query.

However, correlated subqueries are not supported by Dari.

So something like:
```
SELECT name, organization
  FROM employee AS outer
  WHERE salary > (
    SELECT AVG(salary)
      FROM employee
      WHERE organization = outer.organization);
```
cannot be done with the query API. You can probably get around that by making several, separate queries.

#### Deleting

When you delete a Java class, the data for that class still persists, so you have to delete the data manually:

```
Query.from(Article.class).deleteAll();
```

If the class is already gone, you can use the type ID:

```
Query.fromType(ObjectType.getInstance(uuid)).deleteAll(); // ... ?
```

## The Database

When you add the `@Indexed` annotation to a model, it has to be re-indexed before existing data will be actually indexed. Changing the model only ensures data saved after the index annotation was added will be indexed. You can use the index tool of the db-bulk dari tool to re-index huge amounts of data efficiently (faster than re-saving those data).

You can access the save functionality that is normally done behind-the-scenes on the Content and Record classes, to manually update the State.


### Embedded

You can use `@Recordable.Embedded` above the class definition to make the class an embedded model, meaning it's defined within the context of another model, i.e. that the data is stored within the other model it's embedded in.

```
// Address.java

@Recordable.Embedded
public class Address extends Content {
  private String street;
  @Indexed

}

// Employee.java

public class Employee extends Content {
  @Indexed
  private Address homeAddress; // the address is saved not as a standalone thing, but instead within the Employee Content instance
}

```

Even embedded objects within a model have their own ID values to track unique values, e.g. to keep the order of those data straight.

### Object Types

[ObjectType API Documentation](https://artifactory.psdops.com/psddev-releases/com/psddev/dari/3.2.2188-2d7dae/dari-3.2.2188-2d7dae-javadoc.jar!/com/psddev/dari/db/ObjectType.html)

"Dari Bootstrap" scans classes for Recordables (e.g. Record and Content class es) to read the classes and generate object types out of each class.
`ObjectType` extends Record with fields like `displayName`, `internalName`, etc. (see more in the dari repo)
`groups` are classes that an Object Type extends or implements (not exactly the hierarchy, which is defined by `java.superClasses`)
`java.assignableClasses` is the list of classes that the ObjectType can be assigned to.
`fields`

### Object Fields

[Object Field API Documentation](https://artifactory.psdops.com/psddev-releases/com/psddev/dari/3.2.2188-2d7dae/dari-3.2.2188-2d7dae-javadoc.jar!/com/psddev/dari/db/ObjectField.html)


### Symbol


symbolId refers to Symbol table, which has a number mapping to a unique index name, e.g.

Symbol ID: `51` Index name: `brightspot.core.Article/embeddedAuthor/name`

The symbol table has references to every indexed field.

### Deprecating

For deprecating fields, see Annotations > `@Deprecated`

The rule is: don't rename classes.

### Transactions

More on [transactions](http://docs.brightspot.com/dari/advanced/database-transactions.html).

```
static {
  Article a = null;
  Article b = null;

  Database database = Database.Static.getDefault();

  database.beginWrites();

  try {
    a.save();
    b.save();
    database.commitWrites();
  } finally {
    database.endWrites();
  }

}
```

The above is an example where you can prevent the `a` Article from saving if the `b` Article fails to save, so that you don't end up with a partial workflow committed.

### Simulating Publish

Saving from the Java (`.save()`) doesn't mimic an actual publish (so it only sets the field you set and bypasses many Dari workflows).

You can still publish from the Java.
```

Article a;



Content.Static.publish(a, null, null);
```

## Annotations

### `@Deprecated`

`@Deprecated` over a field in a class model, the field will show if data exists for it, otherwise it will hide that field (e.g. if a new instance is created).

```
@Deprecated
private String title;

private String headline; // to replace title
```

You can use `beforeSave()` to null out the deprecated field and save the existing data as the new field, e.g. copy `title` to `headline`.

You can also override the getter for the new field, e.g. `getHeadline()` to use the deprecated field if the new field is `null`.

You can try saveUnsafely to skip validation and checks and quickly update instances to remove the deprecated field (and add the new field, depending on what's necessary).

This is trickier with Indexing. You can always try to change the display name, instead of deprecating the old field, e.g.

```
@Indexed
@DisplayName("Headline") // instead of deprecating, just change the editor-facing name.
private String title;
```

If you change the type of an existing field to a type that existing data cannot convert to, it produces data called `dari.trash` which is only accessible through the State (it's not stored in the MySQL until published after the trash is produced).

[Dari Trash API Documentation](https://artifactory.psdops.com/psddev-releases/com/psddev/cms/3.2.5745-1cb7d2/cms-3.2.5745-1cb7d2-javadoc.jar!/com/psddev/cms/db/Trash.html)

For example, let's say you change as such:

```
// private String title; // original field, with data like "Hello World!"
private Integer title; // new version - not compatible to convert Strings to Integers
```

### `@InternalName`

`@InternalName` overrides what the back-end name is used for a given property, so it's the inverse of using `@DisplayName`:
You change the property in the Java class, but tell Dari to use a different property (by using InternalName), rather than keeping the same property in the Java class and just changing the way it looks to editors (DisplayName).

e.g. need to use `headline` instead of `title`:

```
// private String title; // deprecated, needs to be replaced by headline

@InternalName("title") // this tells Dari to refer to title rather than headline
private String headline; // this replaced the title private property
```


### `@Recordable.DisplayName`

- `@Recordable.DisplayName("BLOG ARTICLE")` - to change the display name for an ObjectType

### `@Indexed(visibility = true)`

... something about making fields invisible, for example when some data is in a trashed state.

### `@Recordable.FieldInternalNamePrefix`

Used to ensure a unique name by appending the prefix, generally (such as with modification pattern) by the name of the common interface, e.g. `@Recordable.FieldInternalNamePrefix("durationTrackable.")`

### `@ToolUi.Hidden(false)` for Interfaces

Use the above to make sure an interface can be searched by editors.

## Validating Content

Let's say you want to enforce two authors for an article.

In Article.java:

```
@Indexed
@ToolUi.Note("Minimum of two authors required.");
private List<Author> authors;

// ... then, down below

@Override
public void beforeSave() {
  if (authors == null || authors.size() < 2) {
    getState().addError(
      getState().getField("authors"), "Validation message: Two authors are required, at least."));
  }
}
```

The `beforeSave()` method uses State to add a validation message when there is an error.

### The Methods


These happen in order:

```

@Override
public void beforeSave() {
  // will impact response time of the UI - this runs before the save happens
  // don't query here, or do other calls that take time
}

@Override
public void onValidate() {

}

@Override
public void beforeCommit() {

}

@Override
public Boolean onDuplicate(ObjectIndex index) {
  // handle duplication logic, e.g.
  if (HEADLINE_FIELD.equals(index.getName())) {
    headline = headline + "_1"; // this identifier is preferably not possibly a duplicate
  }

  return super.onDuplicate(index);
}

@Override
public void afterSave() {
  // don't do anything with the database here...
}
```

You can find these methods here:
[ToolUi API Documentation](https://artifactory.psdops.com/psddev-releases/com/psddev/cms/3.2.5745-1cb7d2/cms-3.2.5745-1cb7d2-javadoc.jar!/com/psddev/cms/db/ToolUi.html)

## Field Level Locking

[Field Locking Documentation](http://docs.brightspot.com/cms/editorial-guide/locking/all.html)

When an editor clicks into a field to edit, that particular field is locked for other editors modifying the same piece of Content (it greys out and can't be modified, and has a message about being locked).

## Drafts

The "Initial Draft" is the first draft saved.
The "Scheduled Draft" ...
"Scheduled" ...

## Save APIs

```
save();

saveUniquely();

saveImmediately(); // save independent of the transaction (skip the transaction control)

// same as save() for MySQL, but for Solr there are two steps:
// 1. posting document and saving it
// 2. then making it available to search (which takes longer)
// saveEventually() lets it save it in Solr but not do the slower search index right then, for performance optimization
saveEventually();

// skips before and after saves and all validation steps (e.g. unique constraint)
getState().saveUnsafely(); // intentionally abstracted to be harder to use, by requiring getting the state first before using it

```

## Links to Resources

- [Dari Repository](https://github.com/perfectsense/dari)
- [API documentation root](https://artifactory.psdops.com/psddev-releases/com/psddev/dari-db/3.2.2448-622660/dari-db-3.2.2448-622660-javadoc.jar!/index.html)
- [ObjectType API Documentation](https://artifactory.psdops.com/psddev-releases/com/psddev/dari/3.2.2188-2d7dae/dari-3.2.2188-2d7dae-javadoc.jar!/com/psddev/dari/db/ObjectType.html)
- [Query API reference](https://artifactory.psdops.com/psddev-releases/com/psddev/dari/3.2.2188-2d7dae/dari-3.2.2188-2d7dae-javadoc.jar!/com/psddev/dari/db/Query.html)
- [Schedule API Documentation](https://artifactory.psdops.com/psddev-releases/com/psddev/cms/3.2.5745-1cb7d2/cms-3.2.5745-1cb7d2-javadoc.jar!/index.html)
- [Field Locking Documentation](http://docs.brightspot.com/cms/editorial-guide/locking/all.html)
- [ToolUi API Documentation](https://artifactory.psdops.com/psddev-releases/com/psddev/cms/3.2.5745-1cb7d2/cms-3.2.5745-1cb7d2-javadoc.jar!/com/psddev/cms/db/ToolUi.html)
