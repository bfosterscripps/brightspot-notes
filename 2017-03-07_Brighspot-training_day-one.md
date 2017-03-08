# Brightspot Training: Day One

Rebecca McNeilly - rmcneilly@perfectsensedigital.com - Account Manager

Dan Slaughter - Brightspot architect - trainer

## Agenda

9:30 - LL Training Room
  - instruction and set-up

10:00 - LL Training Room
  - Hello World Intro to Brightspot
  - BSP End to End development environment setup
  - Quick installation
  - Simple Data Model

12:00 - Cellar Cafeteria
  - Lunch

13:00 - East / West Conference Rooms
  - Break

14:30 - East / West Conference Rooms
  - Training Continued
  - Simple Template
  - Bind Data Model and Template Together

16:30 - Close, End of Day



```
git clone git@github.com:perfectsense/brightspot-tutorial.git
cd brightspot-tutorial
git checkout init
bash run.sh
```

It will pop up http://localhost:9480/cms/index.jsp
Login to local with admin/admin.

The `run.sh` is using a tiny database, `m2`, which will get Brightspot put up really quickly. Vagrant on the other hand will setup a full-blown instance of Brightspot.

Brightspot runs in any servlet container - generally Tomcat.
Brightspot typically uses MySQL and Solr.
S3 is typically used.

http://docs.brightspot.com/cms/editorial-guide/glossary.html

[Dari](https://github.com/perfectsense/dari) is the back-end - a library of tools for storing and modeling data.

[Brightspot](https://github.com/perfectsense/brightspot-cms) is a website built on Dari which allows you to build other websites.
In addition, it's meant to be a clean slate for defining the nature of your data, and to be extensible.

Dari ends up being a few jar files that are run by Brightspot

Typical Maven project:
src/main/java - BE assets
src/main/webapp - FE assets
pom.xml - for building

To install Vagrant and Virtualbox with homebrew:

```
brew cask update
brew install Caskroom/cask/virtualbox
brew install Caskroom/cask/vagrant
```

(See [here](https://github.com/danslaughter2/training/blob/master/intro.md))


To get Vagrantfile:
https://github.com/danslaughter2/training/blob/master/Vagrantfile

Save the file at the root of the repo: `brightspot-tutorial/Vagrantfile`

```
vagrant up # start up vagrant
vagrant ssh # login to the Vagrant box once it's startup
sudo -i # login as root
ifconfig # get your IP address
# at this point, brightspot won't be working
# we need to copy the war file for Brightspot into the webapp dir
cp /vagrant/target/tutorial-1.0-SNAPSHOT.war /servers/training/webapps/ROOT.war # change out the name of the war and target dir name as needed
```

Once you have copied the war to the webapps dir, open up the CMS:
http://172.28.128.15/cms (use the IP you got above from the `ifconfig`)
(login with admin:admin)



```
### These are ways to manage the services:
service solr start|stop|restart
service $projectName start|stop|restart # project name in the training exercise is "training", but it will vary (and is defined in your Vagrantfile)
service mysql start|stop|restart
service apache start|stop|restart
```

use `ifconfig`, get your IP address ($IP)
Solr: $IP:8180/solr
Apache: $IP:80/dims-status/ (reverse proxy) - needs the last slash
e.g. this URL: http://172.28.128.15/dims-status/ should produce something like:

```
ALIVE

Uptime:  12 minutes 25 seconds
Restart time: Tuesday, 07-Mar-2017 10:36:35 EST

mod_dims version: 3.3.9 ($Revision: $)
ImageMagick version: ImageMagick 6.9.2-3 Q8 x86_64 2016-01-11 http://www.imagemagick.org
libcurl version: libcurl/7.35.0 OpenSSL/1.0.1f zlib/1.2.8 libidn/1.28 librtmp/2.3

Details
-------
Successful requests: 1
Failed requests: 0

Download timeouts: 0
Imagemagick Timeouts: 0
```

image processing is done by [dims](https://github.com/beetlebugorg/mod_dims)
It was created by an ops guy named Jeremy while he was at AOL, it was open-sourced and is now used by PSD.

database files live in
/var/lib/$serviceName # $serviceName for example solr or mysql

/vagrant is a link to your repo root
/servers/$projectName # where $projectName would be something like "training", see above


src/java/brightspot.core/Article.java :

```
package brightspot.core;

import com.psddev.cms.db.Content;

// Recordable (interface) - defined in Dari
// Record (abstract class) - Dari abstract class, can be used by Brightspot, but it won't show up by default to editors in Brightspot
// Content (abstract class) - accessible in Brightspot, content type visible to editors

public class Article extends Content {
  private String title;

}
```

URLs:
tutorial.vagrant/cms - Brigtspot CMS
tutorial.vagrant/_debug - Dari utilities

/cms/admin/users.jsp

## The View Model

[Article Develop View Model Tutorial](http://docs.brightspot.com/cms/developers-guide/tutorials/article-develop-view-model.html)

src/java/brightspot.core/ArticleViewModel.java:

```
package brightspot.core;

import com.psddev.cms.view.JsonView;
import com.psddev.cms.view.ViewInterface;
import com.psddev.cms.view.PageEntryView;
import com.psddev.cms.view.ViewModel;

@JsonView
// @HandlebarsTemplate("/path/to/template") // instead of a JsonView, use a Handlebar template to render
@ViewInterface // uses Java bean spec to figure out the data elements are on the class,
// i.e. it looks for all the getter methods and generates a model based on those methods,
// e.g. `foo()` method would be ignored, but `getBar()` would be turned into a property of `bar` with the value returned by that method.
public class ArticleViewModel extends ViewModel<Article> implements PageEntryView {
  public String getHeadline() {
    return model.getTitle(); // the act of specifying Article in ViewModel<Article> gives us access to the Article.java title
  }

  public foo() {
    return model.getTitle();
  }

  public getBar() {
    return model.getTitle();
  }

}
```

but, if you don't use the ViewInterface annotation but implement a View that does:

src/java/brightspot.core/ArticleView.java:

```
import com.psddev.cms.view.ViewInterface;
public class ArticleView extends ViewModel<Article> implements PageEntryView {

}
```

(modified to not use @ViewInterface annotation)
src/java/brightspot.core/ArticleViewModel.java:

```
package brightspot.core;

import com.psddev.cms.view.JsonView;
import com.psddev.cms.view.ViewInterface;
import com.psddev.cms.view.PageEntryView;
import com.psddev.cms.view.ViewModel;

@JsonView
// @HandlebarsTemplate("/path/to/template") // instead of a JsonView, use a Handlebar template to render
@ViewInterface // uses Java bean spec to figure out the data elements are on the class,
// i.e. it looks for all the getter methods and generates a model based on those methods,
// e.g. `foo()` method would be ignored, but `getBar()` would be turned into a property of `bar` with the value returned by that method.
public class ArticleViewModel extends ViewModel<Article> implements PageEntryView {
  public String getHeadline() {
    return model.getTitle(); // the act of specifying Article in ViewModel<Article> gives us access to the Article.java title
  }

  public foo() {
    return model.getTitle();
  }

  public getBar() {
    return model.getTitle();
  }

}
```


src/java/brightspot.core/ArticleViewModel.java:

```
package brightspot.core;

import com.psddev.cms.view.JsonView;
import com.psddev.cms.view.ViewInterface;
import com.psddev.cms.view.PageEntryView;
import com.psddev.cms.view.ViewModel;

@HandlebarsTemplate("/brightspot/core/Article") // uses the article handlebar template
@ViewInterface
public class ArticleViewModel extends ViewModel<Article> implements PageEntryView {
  public String getHeadline() {
    return model.getTitle(); // the act of specifying Article in ViewModel<Article> gives us access to the Article.java title
  }

  public String getDescription() {
    return "Hardcoded description";
  }

}
```

[Article Develop View Tutorial](http://docs.brightspot.com/cms/developers-guide/tutorials/article-develop-view.html)

src/webapp/brightspot/core/Article.hbs:

```
<div class="Article">
  <h1 class="Article-headline">
    Headline Text
  </h1>

  <div class="Article-body">
    Body Paragraphs
  </div>
</div>
```

...

src/java/brightspot.core/Author.java:

...

`import com.psddev.dari.util.StorageItem` - file that can be stored, e.g. image


## Dari Tools

[Dari Tools Documentation](http://docs.brightspot.com/cms/developers-guide/dari-tools/all.html)

tutorial.vagrant/_debug - Dari utilities

[Schema Viewer Dari Documentation](http://docs.brightspot.com/dari/data-modeling/schema-viewer.html)
Database Schema Viewer: shows the classes for the content types, e.g. brightpot.core.Article

`ToolUser user = this.getPublishUser();`

ToolUser is also in the Database Scheme Viewer - it will show the relationship between that class and others (think database UML)

Dari Query tool can be used to search for instances of a class


Dari Profile Result - showing all that has happens - data base reads, SQL queries, resolving URLs, rendering with Handlebar templates
you can see the lines of code being executed here, which is also logged so that problems in the log are traced back directly to code


can use Dari Code Editor to modify the handlebar template

Reloader doesn't pick up new classes until a structural change has been made to that class after reloading. you can also use a query parameter of `_reload=true` to force it to reload.

## Front-End

[Brightspot Styleguide Repository](https://github.com/perfectsense/brightspot-styleguide)

where the js and CSS live


## Kitchen Sink

RTE Inline (just bold, other styles for inline elements)
RTE Block (allows for lists and other more complicated HTML)
RTE Custom (define your own toolbar)

RichText is Brightspot specific class, but Number, Boolean, Locale (as of Java 8), Date, and the rest are standard Java.
Spatial Java classes are Brightspot specific:
- Region: draw a region on a map
- Location: drag a point on a map

Region still points to MapQuest, but needs to use OpenStreet map point. This is a known bug and should be fixed shortly.

## Annotations

[Dari Annotation Documentation](http://docs.brightspot.com/dari/reference/annotations.html)

click the ? > String Field > For Developers tab > "Possible Annotations"

extensible - can create new annotations and modify existing


"dynamic" NoteHtml
right now it looks like: `@ToolUi.NoteHtml("<span data-dynamic-html='${content.getField3NotHtml()}'></span>")`
but at some point it should be something like: `@ToolUi.DynamicNoteHtml("getField3NotHtml")`
the getField3NotHtml() would then be defined, see example:

```
@ToolUi.NoteHtml("<span data-dynamic-html='${content.getField3NotHtml()}'></span>")
private String field3;

public String getField3NotHtml() {
  return String.valueOf(new Random().newInt());
}
```

Instead of `content` in the dynamic NoteHtml annotation, you could use `field` or `toolPageContext` (more on that later).

You could change `getField3NotHtml()` to something like...

```
public String getField3NotHtml() {
  return field1 + " - " + field2;
}
```

and as you type and change field1 or field2, the field3 NoteHtml dynamically updates to the new values.

`@ToolUi.Placeholder("This is a placeholder value.")` - greyed out text in the box, like HTML5 placeholder (just informational)

`@Required` automatically puts a placeholder of `(Required)`

`@ToolUi.Placeholder("This is a placeholder value.")` is short for `@ToolUi.Placeholder(value = "This is a placeholder value.")`
More interesting is: `@ToolUi.Placeholder(dynamicText = "${content.getSomeValue()}", editable = true)`

the `editable = true` causes the placeholder to become the value of the field upon scolling over it (popping into place).

Adding annotation or changing body of method, without structural changes to the class the recompile isn't necessary.
full reload is required when there have been structural changes

`@Indexed` makes it so that field can be queried by

"The Label" is what editors see in the interface. You can override that:
```
@Override
public String getLabel() {
  return getName();
}
```

```
@Recordable.LabelFields({"name"}) // as a property name
// @Recordable.LabelFields({"getName"}) // as a method
public class ...
```

## Query

[Dari Querying Documentation](http://docs.brightspot.com/dari/query/index.html)

```
@Indexed
public List<Team> getTeams() {
  return Query.from(Team.class).where("employees = ?", this).selectAll();
}
```


## Links to Resources

- [Brightspot Documentation](http://docs.brightspot.com)
- [Training resources](https://github.com/danslaughter2/training)
- [Dari Querying Documentation](http://docs.brightspot.com/dari/query/index.html)
- [Dari Annotation Documentation](http://docs.brightspot.com/dari/reference/annotations.html)
- [Brightspot Styleguide Repository](https://github.com/perfectsense/brightspot-styleguide)
- [Schema Viewer Dari Documentation](http://docs.brightspot.com/dari/data-modeling/schema-viewer.html)
- [Dari Tools Documentation](http://docs.brightspot.com/cms/developers-guide/dari-tools/all.html)
- [Article Develop View Tutorial](http://docs.brightspot.com/cms/developers-guide/tutorials/article-develop-view.html)
- [Article Develop View Model Tutorial](http://docs.brightspot.com/cms/developers-guide/tutorials/article-develop-view-model.html)
