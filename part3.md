## Partie 3: Gérer la persistance avec Room 
Dans cette section, nous allons utiliser une base de données pour gérer la persistance afin que les données persistent au redemarrage de l'application. 

Le schéma suivant représente l'architecture du code de l'application 
![Architecture](/architecture.png "Nouveau voisin")

### Mise en place de la base de données avec Room
1. Dépendances 
Modifier le fichier ```build.gradle.kts``` du module de l'application.

```gradle
   implementation("androidx.room:room-runtime:2.6.1")
    annotationProcessor("androidx.room:room-compiler:2.6.1")

    // To use Kotlin Symbol Processing (KSP)
    ksp("androidx.room:room-compiler:2.6.1")

    // optional - Kotlin Extensions and Coroutines support for Room
    implementation("androidx.room:room-ktx:2.6.1")
```
> Ne pas oublier de resynchroniser gradle

2. Dans le package ``dal`` ajouter un package ``local`` puis ajouter les sous-package suivants : ``entities``, ``daos``

3. Dans le package ``local``, ajouter une classe ``NeighborEntity`` qui permet de modéliser un objet ``neighbor`` dans la base de données

```Kotlin
@Entity(tableName = "neighbors")
class NeighborEntity(
    @PrimaryKey(autoGenerate = true)
    val id: Long,
    var name: String,
    var avatarUrl: String,
    val address: String,
    var phoneNumber: String,
    var aboutMe: String,
    var favorite: Boolean = false,
    var webSite: String? = null,
)
```

3. Dans le package ``daos``, Ajouter une interface ``NeighborDao``.

4. Modifier l'interface ```NeighborDao``` pour y ajouter une méthode pour récupérer la liste des neighbors dans la base de données 

```Kotlin
@Dao
interface NeighborDao {
    @Query("SELECT * from neighbors")
    fun getNeighbors(): LiveData<List<NeighborEntity>>
}
```

5. Dans le package ```local```, ajouter une classe ```NeighborDataBase``` permettant de modéliser la base de données. 

```Kotlin
@Database(
    entities = [NeighborEntity::class],
    version = 1
)
abstract class NeighborDataBase : RoomDatabase() {
    abstract fun neighborDao(): NeighborDao

    companion object {
        private var instance: NeighborDataBase? = null
        fun getDataBase(application: Application): NeighborDataBase {
            if (instance == null) {
                instance = Room.databaseBuilder(
                    application.applicationContext,
                    NeighborDataBase::class.java,
                    "neighbor_database.db"
                )
                    .fallbackToDestructiveMigration()
                    .build()
            }
            return instance!!
        }
    }
}

```

<details>

<summary> Pour mieux comprendre : </summary>
- L'annotation ```@Database``` permet de définir les entités et la version de la base de données. 

- Une base de données relationnelle peut contenir plusieurs tables, par conséquent l'attribut entities de l'annotation permet de définir les entités gérées par la base de données. 

- Le numéro de version est très important car il permet d'indiquer à Room quand le schéma de la BDD change. Ainsi, à chaque modification du modèle, il faut incrémenter le numéro de version. 

- La classe qui modélise la base de données doit être abstraite, Room générera l'implémentation. 

- Les DAOs sont référencés comme des méthodes abstraites dans la classe modélisant la base de données. Les implémentations seront générées par Room. 

- Finalement, la base de données doit être un singleton. Cette approche est importante car elle permet d'avoir un seul point d'entrée sur la base de données.

- fallbackToDestructiveMigration() indique à Room de supprimer la base de données et la recréer quand le modèle de données change. 

</details>

### Implémenter une nouvelle source de données 
Dans cette section, nous allons définir une classe qui permet d'exposer les données de la base de données selon le protocole défini par l'interface ```NeighborService```.

1. Dans le package ```local```, ajouter une classe ```LocalNeighbor``` qui implémente l'interface ```NeighborService```

```Kotlin

class LocalNeighbor(application: Application) : NeighborDatasource {
    private val database: NeighborDataBase = NeighborDataBase.getDataBase(application)
    private val dao: NeighborDao = database.neighborDao()

    private val _neighors = MediatorLiveData<List<Neighbor>>()
    
    init {
        
        _neighors.addSource(dao.getNeighbors()) { entities ->
            _neighors.value = entities.map { entity ->
                entity.toNeighbor()
            }
        }
    }

    override val neighbours: LiveData<List<Neighbor>>
        get() = _neighors

    ...
}

```

<details>

<summary>Petite analyse </summary>

- Cette classe implémente ```NeighborService``` qui l'oblige à avoir le même comportement que la classe ```InMemoryNeighbor```. Ainsi, la seule différence entre les deux sources de données est le mode de persistance. 

- Ici on a utilisé un ```MediatorLiveData``` qui nous permet d'observer les changements sur la base de données et appliquer des modifications sur les données renvoyées par la BDD avant de les afficher à l'utilisateur. 

</details>

## Convertion de données 
<details>
<summary>Explications</summary>
A ce stade, la base de données à son modèle de données qui est différente de celle de la vue. Plus tard, quand on va rajouter des web services, eux aussi vont avoir leur propre modèle de données. Un des principes du clean architecture consiste à utiliser un modèle dédié pour chaque couche et de définir des routines de conversion quand il faut faire passer des données d'une couche à l'autre. 
</details>

1. Dans le package ```dal```, ajouter un nouveau package ```utils``` puis un fichier Kotlin ```NeighborMapper.kt```. 

2. Créer une fonction d'extension dans ```NeighborMapper.kt``` permettant de convertir une instance de ```NeighborEntity``` en ```Neighbor```

> Rappelez-vous, les fonctions d'extension permettent d'implémenter une fonction dans une classe sans la surcharger. 

```Kotlin
fun NeighborEntity.toNeighbor() = Neighbor(
    id = id,
    name = name,
    avatarUrl = avatarUrl,
    address = address,
    phoneNumber = phoneNumber,
    aboutMe = aboutMe,
    favorite = favorite,
    webSite = webSite ?: ""
)
```

## Adapter le repository 
La data source ```LocalNeighbor``` a besoin d'une instance de l'application pour s'initialiser; en fait c'est la base de données qui en a besoin. 
Pour cela, nous allons modifier le repository pour qu'elle accepte en paramètre une instance d'application. 

1. Modifier le repository

```Kotlin
class NeighborRepository private constructor(application: Application) {
    private val dataSource: NeighborService

    init {
        dataSource = LocalNeighbor(application)
    }

    companion object {

        @Volatile private var instance: NeighborRepository? = null

        fun getInstance(application: Application) =
            instance ?: synchronized(this) { 
                instance ?: NeighborRepository(application).also { instance = it }
            }
    }
}
```

<details>

<summary>On essaie de comprendre ? </summary>

- On veut maitriser la création du repositoty, on met son constructeur en private.

- On veut que le reposirory soit un singleton, on crée une méthode statique pour l'initialiser et on rend son constructeur private. Ainsi, on ne pourra pas l'instancier à lextérieur de la classe. 

- Finalement, on change de source de données; on remplace ```InMemoryNeighbor``` par ```LocalNeighbor``` 

</details>

## Mise en place de l'injection de dépendance
Une des bonnes pratiques dans la clean architecture est l'utilisation de l'injection de dépendance. Ce mécanisme permet de gérer la création des instances d'objets partagés comme les repositories. 

Nous allons implémenter un mécanisme d'injection de dépendance mais vous pouvez utiliser des librairies spécialiées (comme HILT) qui le font plus proprement. 


1. Dans le package principale, ajouter un package ``di``

2. Dans ``di``, ajouter une classe Injection

```kotlin
object Injection {
    private var repository: NeighborRepository? = null

    fun init(application: Application) {
        repository = NeighborRepository(application)
    }

    fun getRepository(): NeighborRepository {
        return repository!!
    }
}
```

3. Modifier l'activité pour initialiser l'injecteur

```kotlin

override fun onCreate(savedInstanceState: Bundle?) {
    ...
    Injection.init(application)
}

```

4. Modifier les VMs pour récurer l'instance du repository en passant par l'injecteur 


5. Ajouter les voisins de tests dans la BDD

Afin d'éviter d'avoir une base de données vide, nous allons modifier la classe qui gère la base de données pour y insérer des données de tests à la création de la base de données. 

```Kotlin
fun getDataBase(application: Application): NeighborDataBase {
            if (instance == null) {
                instance = Room.databaseBuilder(
                    application.applicationContext,
                    NeighborDataBase::class.java,
                    "neighbor_database.db"
                ).addCallback(object : Callback() {
                    override fun onCreate(db: SupportSQLiteDatabase) {
                        super.onCreate(db)
                        insertFakeData()
                    }
                })
                    .fallbackToDestructiveMigration()
                    .build()
            }
            return instance!!
        }

        private fun insertFakeData() {
            Executors.newSingleThreadExecutor().execute {
                InMemory_NeighborS.forEach {
                    instance?.neighborDao()?.add(it.toEntity())
                }
            }
        }
```

6. Compiler et tester


## A vous de jouer

1. Modifier l'écran d'ajout pour sauvegarder les nouveaux voisins dans la base de données

2. Implémenter les actions de suppression, d'ajout en favoris et de like de voisins. 

> ATTENTION, vous n'avez pas besoin de raffraichir la liste, elle sera mise à jour automatiquement via le live Data. 

3. Ajouter un menu dans la toolbar permettant à l'utilisateur de choisir la source de données à utiliser : 
- Non persistent --> InMemory
- Persistent --> Local

4. Ajouter un menu dans la toolbar permettant de filtrer la liste en fonction des critères (tous, favoris, liked). Ce menu doit s'afficher uniquement dans le fragment de la liste 

