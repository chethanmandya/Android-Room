
#### What is Room ? 
Room is a persistence library, it is part of jetpack. library takes care most of complicated stuff that we previously had to do ourselves, we will write much less boilerplate code to create tables and make database operations.

It makes it easier to work with SQLiteDatabase objects in your app, decreasing the amount of boilerplate code and verifying SQL queries at compile time


### Sqlite in android is not that cool 
- You need to write out a biolerplate code to convert between your java object and your sqlite object. 
- It doesn't have compile time safety, if you building sqlite query and if you forgot to add comma, you going to get run time crash, that makes you very hard to test all those cases you put. 
- When you are writing reactive application and you want to observe the databases changes to UI , sqlite doesn't facilate to do that. 

I am not going to go too much on theoritical knowledge, if you have already used any of those sql wrapper like ORMLight, Realm, you will understand advantagious and disadvantgious of having Room over any other library. Let me step into an example where you can understand how to use room and its feature. 

There are 3 major components in Room:
 - Database: Contains the database holder and serves as the main access point for the underlying connection to your app's persisted, relational data.
 - Entity: Represents a table within the database.
 - DAO: Contains the methods used for accessing the database.


Below example is json response of near by venues which are available on foresquare apis. Assume your response look like as below. 

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
 
It is really not necessary to have all of the information of venue object which comes venue response, Creating a User Minimal object that holds only the data needed will improve the amount of memory used by the app. it is always recommended to load only the subset of fields what is needed for UI, that will improve the speed of the queries by reducing the IO cost. Hence I have consider below fields in the venue table.

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

Below code snippet shows how to define an Dao class for your venu entity

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

When you annotated field as Embedded, all of those nested field of annotated field will be referred as separate column in the same Entity. 

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

if suppose, your field has array list as one of the sub field then you will save this field either by foreign key relation OR by type converters. 

you will go for making it as foreign key relation when it has very complex structure, structure which has nested list within list. or else you go for type convertor when it has only list of objects, like list of premitive type. 

In the below example, VenuDetails has a field called Photos, The Photos has nested list, it is a relation 1 to Many. To map this type of relation we will use the @ForeignKey annotation. 

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
Below entity of VenuePhotos saves the Photo object information which we have igrnored in VenueDetails. You can have your own version of entity to save Photo object element like "url". it is really not necessary to have complete Photo object with all other field when you are not using in the app. you can see we have consider parent entity as VenuDetails and child entity as VenuePhotos. we are linking by using parent column id in the Venue and Child column venuId. 

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
```











