
#### What is Room ? 
Room is a persistence library, it is part of jetpack. library takes care most of complicated stuff that we previously had to do ourselves, we will write much less boilerplate code to create tables and make database operations.

It makes it easier to work with SQLiteDatabase objects in your app, decreasing the amount of boilerplate code and verifying SQL queries at compile time


### Sqlite in android is not that cool 
- You need to write out a boilerplate code to convert between your java object and your sqlite object. 
- It doesn't have compile time safety, if you building sqlite query and if you forgot to add comma, you going to get run time crash, that makes you very hard to test all those cases you put. 
- When you are writing reactive application and you want to observe the databases changes to UI , sqlite doesn't facilitate to do that. 

I am not going to go too much on theoretical knowledge, if you have already used any of those sql wrapper like ORMLight, Realm, you will understand the advantages and disadvantages of having Room over any other library. Let me step into an example to make you understand how to use the room and its features. 

There are 3 major components in Room:
 - Database: Contains the database holder and serves as the main access point for the underlying connection to your app's persisted, relational data.
 - Entity: Represents a table within the database.
 - DAO: Contains the methods used for accessing the database.


Below example is json response gives you nearby venues which are available on foursquare apis. Assume your response look like as below. 

```kotlin
 "venues": [
      {
        "id": "5a2285eddee7701b1d63d2d3",
        "name": "Trainmore",
        "location": {
          "address": "Coolsingel 63",
          "lat": 51.92291909950766,
          "lng": 4.478042374114597,
          "labeledLatLngs": [
            {
              "label": "display",
              "lat": 51.92291909950766,
              "lng": 4.478042374114597
            }
          ],
          "postalCode": "3012 AS",
          "cc": "NL",
          "city": "Rotterdam",
          "state": "South Holland",
          "country": "Netherlands",
          "formattedAddress": [
            "Coolsingel 63",
            "3012 AS Rotterdam",
            "Netherlands"
          ]
        },
        "categories": [
          {
            "id": "4bf58dd8d48988d175941735",
            "name": "Gym \/ Fitness Center",
            "pluralName": "Gyms or Fitness Centers",
            "shortName": "Gym \/ Fitness",
            "icon": {
              "prefix": "https:\/\/ss3.4sqi.net\/img\/categories_v2\/building\/gym_",
              "suffix": ".png"
            },
            "primary": true
          }
        ],
        "referralId": "v-1557410027",
        "hasPerk": false
      }]
      
```

#### @Entity : 
Room creates a table for each class annotated with @Entity; the fields in the class correspond to columns in the table.
 
Keeping only what we needed
 
It is really not necessary to have all of the information of venue object which comes venue response, Creating a User Minimal object that holds only the data needed will improve the amount of memory used by the app. it is always recommended to load only the subset of fields what is needed for UI, that will improve the speed of the queries by reducing the IO cost. Hence I have considered below fields in the venue table.

The following code snippet shows how to define an entity:

``` kotlin
@Entity(
    indices = [
        Index("location_city")],
    primaryKeys = ["id"]
)
data class Venue(
    @field:SerializedName("id")
    var id: String,

    @field:SerializedName("name")
    var name: String? = "",

    @field:SerializedName("location")
    @field:Embedded(prefix = "location_")
    var location: Location


) : Serializable {

}
```


### @Dao 
DAOs are responsible for defining the methods that access the database.
Below code snippet shows how to define a Dao class for venu entity

```kotlin
@Dao
@OpenForTesting
abstract class VenueDao {


    @Insert(onConflict = OnConflictStrategy.REPLACE)
    abstract fun insertVenue(vararg repos: Venue)


    @Insert(onConflict = OnConflictStrategy.REPLACE)
    abstract fun insertVenues(repositories: List<Venue>)


    @Insert(onConflict = OnConflictStrategy.REPLACE)
    abstract fun insert(result: VenuesSearchResult)


    @Insert(onConflict = OnConflictStrategy.IGNORE)
    abstract fun createVenueIfNotExists(venue: Venue): Long


    @Delete
    abstract fun delete(item: Venue)


    @Query("DELETE FROM Venue")
    abstract fun deleteAll()


    @Query("SELECT * FROM Venue")
    abstract fun loadAllTheVenue(): LiveData<List<Venue>>


    @Query("SELECT * FROM VenuesSearchResult WHERE `query` = :query")
    abstract fun search(query: String): LiveData<VenuesSearchResult>


    fun loadOrdered(repoIds: List<String>): LiveData<List<Venue>> {
        val order = SparseIntArray()
        repoIds.withIndex().forEach {
            order.put(it.index, it.index)
        }
        return Transformations.map(loadById(repoIds)) { repositories ->


            repositories
        }
    }


    @Query("SELECT * FROM Venue WHERE id in (:venueIds)")
    abstract fun loadById(venueIds: List<String>): LiveData<List<Venue>>


    @Query("SELECT * FROM VenuesSearchResult WHERE `query` = :query")
    abstract fun findSearchResult(query: String): VenuesSearchResult?

}
```


#### @Database
To create database we need to define an abstract class that extends RoomDatabase. This class is annotated with @Database, lists of entities contained in the database, and the DAOs which access them. The database version has to be increased by 1, from the initial value.

Below code snippet shows how to define your database class

```kotlin
@Database(
    entities = [
        VenuesSearchResult::class,
        VenueDetails::class,
        VenuePhotos::class,
        Venue::class],
    version = 1,
    exportSchema = false
)

abstract class AppDatabase : RoomDatabase() {
    abstract fun venueDao(): VenueDao
    abstract fun venueDetailsDao(): VenueDetailsDao
}
```

####  @Embedded

When you annotated field as Embedded, all of those nested field of annotated field will be created as a separate column in the same Entity. 

Here location field has `address`, `lat` and `lng` nested field, all of those filed will be created as separated column in same entity Venue. 

```kotlin
@Entity(
    indices = [
        Index("location_city")],
    primaryKeys = ["id"]
)
data class Venue(
    @field:SerializedName("id")
    var id: String,

    @field:SerializedName("name")
    var name: String? = "",

    @field:SerializedName("location")
    @field:Embedded(prefix = "location_")
    var location: Location


) : Serializable {

}

```
#### foreignKeys

if suppose, your field has an array list as one of the sub field then in that case you will save this field either by foreign key relation OR by type converters. 

You will go for making it as foreign key relation when it has very complex structure, structure which has nested list. or you can save them using type convertor when it has only list of objects, like list of primitive type. 

When you have more than one nested list, it is better to save them in foreign key relationship because type convertor is not best suitable, it will slow down performance because of too many traverses in the list while converting user object to primitive and vice versa. 

In the below example, Venue Details has a field called Photos, The Photos has nested list, it is a relation of 1 to Many. To map this type of relation we will use the @ForeignKey annotation. 

``` kotlin 

@Entity(primaryKeys = ["id"])
data class VenueDetails(

        @field:SerializedName("id")
        var id: String,

        @field:SerializedName("name")
        var name: String? = "",

        @field:SerializedName("description")
        var description: String? = "",

        @field:SerializedName("contact") // Nested object
        @field:Embedded(prefix = "contact_")
        var contact: Contact?,

        @field:SerializedName("rating")
        var rating: Double? = 0.0,

        @field:SerializedName("location")
        @field:Embedded(prefix = "location_")
        var location: Location?,

       /**
        we are ignoring field because we going to hold this data by foreigh annotation
       */
        @field:SerializedName("photos")
        @Ignore                                  
        var photos: Photos?

) : Serializable {
    constructor() : this("", "", "", null, 0.0, null, null)

}

```
Below entity of VenuePhotos saves the Photo object information which we have ignored in VenueDetails. You can have your own version of entity to save Photo object element like "url". it is really not necessary to have complete Photo object with all other field when you are not using in the app. you can see in the below snappet we have considered parent entity as Venue Details and child entity as VenuePhotos. we are linking these two entities together by using parent column id in Venue and venueId child column id from VenuePhotos

@Entity(
        indices = [Index("venueId")],
        foreignKeys = [ForeignKey(
                entity = VenueDetails::class,
                parentColumns = ["id"],
                childColumns = ["venueId"],
                onDelete = ForeignKey.CASCADE,
                deferred = true
        )])
data class VenuePhotos(
        @PrimaryKey(autoGenerate = true)
        val id : Int,
        val venueId: String, // this ID points to a VenueDetails
        val url: String? = ""
) {
        constructor(venueId : String, url:String) : this(0,venueId, url)
}

```

#### @TypeConverters

sometimes we may need to store object as is in one column rather than storing them in separate column as in case of @Embedded, so Type Converters comes to the rescue.

Below is the the class which will tell Room how to convert ArrayList object to one of SQLite data type. We will implement methods to convert ArrayList to String for storing it in DB and String back to ArrayList for getting back original User object.

Below is the code snappets where we convert String to Integer list and vice versa. basically table save this data as one of its primitive type rather than user object. 

```kotlin
object VenueTypeConverters {
    @TypeConverter
    @JvmStatic
    fun stringToIntList(data: String?): List<String>? {
        return data?.let {
            it.split(",").map {
                it
            }
        }?.filterNotNull()
    }

    @TypeConverter
    @JvmStatic
    fun intListToString(ints: List<String>?): String? {
        return ints?.joinToString(",")
    }
}


Another example : 

object Converters {
        @TypeConverter
        fun fromString(value: String): ArrayList<String> {
                val listType = object : TypeToken<ArrayList<String>>() {

                }.getType()
                return Gson().fromJson<Any>(value, listType)
        }

        @TypeConverter
        fun fromArrayList(list: ArrayList<String>): String {
                val gson = Gson()
                return gson.toJson(list)
        }
}

```

Public static String fromArrayList(ArrayList<String> list) : 
This method takes our arraylist object as parameter and returns string representation for it so that it can be stored in Room Database.  to make string, just creating Gson object and calling toJson method with our object as parameter is enough.
 
public static ArrayList<String> fromString(String value) : While reading data back from Room Database, we get JSON form of our arraylist which we need to convert back. We will use Gson method fromJson by providing JSON string as parameter. But while converting back, we also need to provide the class of original object (in our case, arraylist), but providing arraylist is not enough here as Gson will not be able know what kind of list it has to form.

