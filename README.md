
#### What is Room ? 
Room is a persistence library and it is part of jetpack.it is wrapper around sqlite that takes care most of complicated stuff that we previously had to do ourselves, we will write much less boilerplate code to create tables and make database operations.


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
Anotation entity create SQLLite table. To save above response, you create an entity as below.
 
 ***NOTE : Keeping only what we needed***
 
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
Data Access Objects are the main classes where you define your database interactions. They can include a variety of query methods. 

Below code snippet shows how to define an Dao class for your venu entity


https://github.com/chethu/Near-by-venus-browsing-sample-with-Android-Architecture-Components/blob/7a08e4c3bd52387c608596f7c89e41b880935b81/app/src/main/java/com/chethan/abn/db/VenueDao.kt#L14-L60


#### @Database




####  @Embedded

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

List of objects can be saved either through foreign key relation OR by type converters,


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

        /*
         Enforce constraints between entities with foreign keys to save photo objects.
         List of objects can be saved either through foreign key relation OR by type converters,
         Photo object json is more lengther and has complex structure, considering Type converter is not
         best approach here, it will slow down performance.
         */
        @field:SerializedName("photos")
        @Ignore
        var photos: Photos?

) : Serializable {
    constructor() : this("", "", "", null, 0.0, null, null)

}




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











