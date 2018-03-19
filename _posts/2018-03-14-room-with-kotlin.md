---
layout: post
title: Room with Kotlin
subtitle: Your data saved in modern way
tags: [kotlin, room, db]
---
[SQLite](https://developer.android.com/training/data-storage/sqlite.html) is a native database of Android. It is widely used in many applications to store data. Other popular solutions are third party solutions like [greenDAO](http://greenrobot.org/greendao), [Realm](https://realm.io/products/realm-database) or [OrmLite](http://ormlite.com).
There is also [Room](https://developer.android.com/training/data-storage/room/index.html). This Android library needs only a bit of configuration. In return it generates all the code necessary to manage SQLite for you.

In this part, we will cover the very bersics to start working with Room. Some other improvements like Transactions, LiveData, ViewModel will come in the other part of _Room with Kotlin_.

### Start with gradle
First make sure that your **project gradle** file has imported Maven repository. Just add google() line.
```gradle
allprojects {
    repositories {
        jcenter()
        google()
    }
}
```

Then in your **app gradle** add these two lines
```gradle
implementation "android.arch.persistence.room:runtime:1.1.0"
kapt "android.arch.persistence.room:compiler:1.1.0"
```
_Check the newest stable version [here](https://developer.android.com/topic/libraries/architecture/adding-components.html)._

{: .box-note}
**Note:** If you use **Java**, you have to change _kapt_ to _annotationProcessor_. Also you don't need to _apply_ it as instructed below.

Make sure you have [kapt](https://kotlinlang.org/docs/reference/kapt.html). Android Studio doesn't provide it automatically. Add this line at the top of gradle
```gradle
apply plugin: 'kotlin-kapt'
```
For more details about adding components [see this developers tutorial](https://developer.android.com/topic/libraries/architecture/adding-components.html).

### All set up. Time to code!
In this example we will create a database with one table representing hats 🎩:  

| id | type | colour_rgb | favourite |
|-------|--------|---------|---------|
| 0 | [Pith helmet](https://en.wikipedia.org/wiki/Pith_helmet) | #FFFFFF | false |
| 1 | [Fedora](https://getfedora.org) | #FF0000 | true |
| 2 | [Top hat](https://en.wikipedia.org/wiki/Top_hat) | #000000 | false |
| ... | ... | ...| ... |

You will need 3 main components. [Database](https://developer.android.com/reference/android/arch/persistence/room/Database.html), [Data Access Object](https://developer.android.com/reference/android/arch/persistence/room/Dao.html), and [Entity](https://developer.android.com/reference/android/arch/persistence/room/Entity.html)

#### Prepare your Model class
Your class needs to be annotaded with ```@Entity```. There has to be at least one field that is a ```@PrimaryKey```. You can add a very useful parameter [autoGenerate](https://developer.android.com/reference/android/arch/persistence/room/PrimaryKey.html#autoGenerate()). When set to true, the primary key will increase automatically. Of course this field has to be a **numeric type**.  

```kotlin
@Entity(tableName = "hat")
data class Hat(@PrimaryKey (autoGenerate = true) val id: Int = 0,
               @ColumnInfo(name = "type")
               var type: String,
               @ColumnInfo(name = "colour_rgb")
               var colourRgb: String,
               @ColumnInfo(name = "favourite")
               var isFavourite: Boolean
               @Ignore
               var isSelected: Boolean = false
)
```
By default the columns will be generated with the properties inherited by the fields. To enforce custom change (like different label or type) use [```@ColumnInfo```](https://developer.android.com/reference/android/arch/persistence/room/ColumnInfo.html).  
_For example you might want to work with Integer, but store it as String. Or you write your code in [camelCase](https://en.wikipedia.org/wiki/Camel_case), but for the database you want to follow [snake_case](https://en.wikipedia.org/wiki/Snake_case)._  
If your class contains some fields which you don't want to store in the database, just mark with ```@Ignore```

#### Go on with DAO
We have our Entity. Now time to do some CRUD.
Create an interface with [```@Dao```](https://developer.android.com/reference/android/arch/persistence/room/Dao.html) annotation. Declare some functions you will use for CRUD.
Annotate them with
* Insert
* Update
* Delete
* Query

This is the place where all the magic happens. During compilation Room will generate an implementation for all necessary database access.

Below I present an example usage.

```kotlin
@Dao
interface HatDao {
    @Insert(onConflict = REPLACE)
    fun insertHats(vararg hat: Hat)

    @Update(onConflict = ROLLBACK)
    fun updateHat(hat: Hat)

    @Delete
    fun deleteHat(hat: Hat)

    @Delete
    fun deleteMultiHats(goneHats: List<Hat>)

    @Query("SELECT * FROM hat")
    fun getAllHats(): List<Hat>

    @Query("SELECT * FROM hat where id = :id")
    fun getHatById(id: Int): Hat
}
```

The arguments of these methods need to be instances of Entity. Or their collections.  ```@Query``` is especially interesting for its flexibility. You can use it to build any SQL query you need. The arguments of this method can be used inside the query.  [Read more](https://developer.android.com/reference/android/arch/persistence/room/Query.html)

There might occur a conflict with one of these constraints: UNIQUE, NOT NULL, CHECK, or PRIMARY KEY. In case of conflicts [SQLite provides 5 different strategies](https://sqlite.org/lang_conflict.html): ABORT (default), ROLLBACK, FAIL, IGNORE, REPLACE.
Set it with  [OnConflictStrategy](https://developer.android.com/reference/android/arch/persistence/room/OnConflictStrategy.html).

Keep creating seperate DAOs for each of your tables.

#### Wrap it up into database
Here comes the last part of our setup. The database class.
You need an abstract class which extends RoomDatabase. Room will generate necessary implementation.
First, take a look at the annotation.
```kotlin
@Database(entities = arrayOf(Hat::class, Shoe::class, Puppy::class), version = 1)
```
It requires an array of entities classes, and the current version. Each entity reflects to a single table.  
Whenever you make changes to your schema, don't forget to raise the version of your database. Otherwise you will be served with a happy ```IllegalStateException```.  

Next, declare here the abstract properties for each of your DAOs. You will use them, to access your perviously declared methods.

```kotlin
abstract val hatDao: HatDao
```
The instance of the database itself you can access using a ```Room.databaseBuilder()```. Just pass context, the class of your database, and a String with its name.

{: .box-note}
**Note:** You can use also ```Room.inMemoryDatabaseBuilder()``` which is perfect for tests. This database will not be saved to the device's storage. You can put there any trash and don't bother to clean it.

It is highly recommended to create an instance of the database as a singleton. Opening a database everytime it is needed might slow down the execution of the code. Also typically there is no need to have more than one database instance at the same time.
Kotlin has a bit different approach for singletons. You can use an _object expression_ to make it work. If you feel like you need to learn a bit more about it - [click here](https://kotlinlang.org/docs/reference/object-declarations.html).

{% highlight kotlin %}
@Database(entities = arrayOf(Hat::class), version = 1)
abstract class HatDatabase : RoomDatabase() {

    abstract val hatDao: HatDao

    companion object {
        private lateinit var INSTANCE: HatDatabase

        fun getInstance(context: Context): HatDatabase {
            synchronized(HatDatabase::class) {
                INSTANCE = Room.databaseBuilder(context.applicationContext,
                        HatDatabase::class.java,
                        context.getString(R.string.hat_database)).build()
            }
            return INSTANCE
        }
    }
}
{% endhighlight %}  


#### OK I have a database. How can I use it?

That is actually a good question. Yet the answer is simple.
1. Get the database instance
2. Access DAO
3. Call a method on DAO

{% highlight kotlin %}
val database = HatDatabase.getInstance(mContext)
val dao = database.hatDao
dao.insertHats(hat1, hat2, hat3)
dao.deleteHat(hat2)
dao.getAllHats()  //returns [hat1, hat3]
{% endhighlight %}

Pretty easy, right? **Not so fast!** Reading/writing data might be time consuming. It is good to introduce some *async task* when you perform these actions. Actually it is required by Room when reading data. If you try to run a ```Query``` method with a ```SELECT``` operation on UI thread, Room will throw ```IllegalStateException``` with the following message: _Cannot access database on the main thread since it may potentially lock the UI for a long period of time_.

So how to run a code asynchronously in Kotlin? The easiest solution is to use [Anko](https://github.com/Kotlin/anko)!
Add it to your project:
```gradle
dependencies {
    compile "org.jetbrains.anko:anko:$anko_version"
}
```
_Check the newest stable version [here](https://plugins.jetbrains.com/plugin/7734-anko-support)._
Then just start another thread in a ```doAsync``` block. To do work on main thread use ```uiThread```.
{% highlight kotlin %}
doAsync {
    mHatsList = dao.getAllHats()
    uiThread {
      mView.displayHats(mHatsList)
    }
{% endhighlight %}

If you want to see a full, working implementation, visit my project [quick notes](https://github.com/supermzn/quick-notes/tree/master/app/src/main/java/com/example/mazena/quicknotes/data).
