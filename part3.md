## Partie 3: Gérer la persistance avec Room 
Dans ce TP, nous allons améliorer l'application Entre Voisin développée dans les parties 1 et 2. 

Jusqu'ici les données de l'application étaient gérées en mémoire, par conséquent quand on relance l'application les données sont réinitialisées. 

Nous allons utiliser une base de données pour gérer la persistance afin que les données persistent au redemarrage de l'application. 

Le schéma suivant représente l'architecture du code de l'application 
![Architecture](/architecture.png "Nouveau voisin")

### Mise en place de la base de données avec Room

Room est un ORM (Object Relational Mapper) qui permet de manipuler les tables d'une base de données SQLite en utilisant uniquement des classes, interfaces et des méthodes. Pour plus d'informations, consultez la documentation ici https://developer.android.com/training/data-storage/room. 

Composants Room : 
**Entity** : Classe permettant de modéliser un objet métier, cette classe sera représentée dans la base de données comme une table. 
Concrètement : 
- Le nom de l'entité représente le nom de la classe
- Les attributs de l'entité représentent les colonnes de la table 
- Les instances de l'entité représentent les lignes de la table. 

**Database** : Classe qui modélise la base de données sous forme d'objets. 

**DAO** : Interface permettant de modéliser les opérations à effectuer sur les entités. Il existe généralement autant de DAO que d'entités.

### 1. Dépendances 
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

### Mise en place de la base de données

1. Dans le package ``dal`` ajouter un package ``local`` puis ajouter les sous-package suivants : ``entities``, ``daos``

2. Dans le package ``local``, ajouter une classe ``NeighborEntity`` qui permet de modéliser un objet ``neighbor`` dans la base de données

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
> Observez bien les variables: 
- id est un val et c'est la clef primaire
- address est un val car on ne veut pas qu'on puisse modifier l'adresse d'un voisin; c'est bête mais vous avez compris l'idée
- website peut être null, tous les voisins ne sont pas des geeks. 
- favorite est une variable non obligatoire, car elle vaut false par défaut et elle est modifiable car on peut ajouter ou enlever un voisin en favori à volonté


3. Dans le package ``daos``, Ajouter une interface ``NeighborDao``.

4. Modifiez l'interface ```NeighborDao``` pour y ajouter une méthode pour récupérer la liste des neighbors dans la base de données 

```Kotlin
@Dao
interface NeighborDao {
    @Query("SELECT * from neighbors")
    fun getNeighbors(): LiveData<List<NeighborEntity>>
}
```

### 4. Modéliser la base de données Room

- Dans le package ```local```, ajouter une classe ```NeighborDataBase``` permettant de modéliser la base de données. 

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

> Pour mieux comprendre : 
- L'annotation ```@Database``` permet de définir les entités et la version de la base de données. 

- Une base de données relationnelle peut contenir plusieurs tables, par conséquent l'attribut entities de l'annotation permet de définir les entités gérées par la base de données. 

- Le numéro de version est très important car il permet d'indiquer à Room quand le schéma de la BDD change. Ainsi, à chaque modification du modèle, il faut incrémenter le numéro de version. 

- La classe qui modélise la base de données doit être abstraite, Room générera l'implémentation. 

- Les DAOs sont référencés comme des méthodes abstraites dans la classe modélisant la base de données. Les implémentations seront générées par Room. 

- Finalement, la base de données doit être un singleton. Cette approche est importante car elle permet d'avoir un seul point d'entrée sur la base de données.

- fallbackToDestructiveMigration() indique à Room de supprimer la base de données et la recréer quand le modèle de données change. 

### 5. Implémenter une nouvelle source de données 
Dans cette section, nous allons définir une classe qui permet d'exposer les données de la base de données selon le protocole défini par l'interface ```NeighborService```.

- Dans le package ```local```, ajouter une classe ```LocalNeighbor``` qui implémente l'interface ```NeighborService```

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

> Petite analyse 

- Cette classe implémente ```NeighborService``` qui l'oblige à avoir le même comportement que la classe ```InMemoryNeighbor```. Ainsi, la seule différence entre les deux sources de données est le mode de persistance. 

- Ici on a utilisé un ```MediatorLiveData``` qui nous permet d'observer les changements sur la base de données et appliquer des modifications sur les données renvoyées par la BDD avant de les afficher à l'utilisateur. 

## Convertion de données 
A ce stade, la base de données à son modèle de données qui est différente de celle de la vue. Plus tard, quand on va rajouter des web services, eux aussi vont avoir leur propre modèle de données. Un des principes du clean architecture consiste à utiliser un modèle dédié pour chaque couche et de définir des routines de conversion quand il faut faire passer des données d'une couche à l'autre. 

- Dans le package ```dal```, ajouter un nouveau package ```utils``` et dans ce nouveau package ajouter un fichier Kotlin ```NeighborMapper.kt```. 

- Créer une fonction d'extension dans ```NeighborMapper.kt``` permettant de convertir une instance de ```NeighborEntity``` en ```Neighbor```

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

> Ce n'est pas très propre de faire cela, on pourrait faire mieux en utilisant des libraires d'injection de dépendances comme Hilt (https://developer.android.com/training/dependency-injection/hilt-android). Pour les plus curieux, ce sera votre point bonus. 

1. Modifier le repository

```Kotlin
class NeighborRepository private constructor(application: Application) {
    private val dataSource: NeighborService

    init {
        dataSource = LocalNeighbor(application)
    }

    // Méthode qui retourne la liste des voisins
    fun getNeighbours(): LiveData<List<Neighbor>> = dataSource.neighbours

    fun delete(neighbor: Neighbor) = dataSource.deleteNeighbour(neighbor)

    companion object {
        private var instance: NeighborRepository? = null

        // On crée un méthode qui retourne l'instance courante du repository si elle existe ou en crée une nouvelle sinon
        fun getInstance(application: Application): NeighborRepository {
            if (instance == null) {
                instance = NeighborRepository(application)
            }
            return instance!!
        }
    }
}
```

> On essaie de comprendre ? 

- On veut maitriser la création du repositoty, on met son constructeur en private.

- On veut que le reposirory soit un singleton, on crée une méthode statique pour l'initialiser et on rend son constructeur private. Ainsi, on ne pourra pas l'instancier à lextérieur de la classe. 

- Finalement, on change de source de données; on remplace ```InMemoryNeighbor``` par ```LocalNeighbor``` 

## Mise en place de l'injection de dépendance
Une des bonnes pratiques dans la clean architecture est l'utilisation de l'injection de dépendance. Ce mécanisme permet de gérer la création des instances d'objets partagés comme les repositories. 
Pour les plus curieux, vous pouvez utiliser des librairies spécialiées (comme HILT) sinon on va créer un injecteur de dépendance (fait maison :) ). 

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

3. Modifier l'activité pour initialiser l'injecteur au lancement de l'activité 

```kotlin

override fun onCreate(savedInstanceState: Bundle?) {
    ...
    Injection.init(application)
}

```

4. Modifier les VMs pour récurer l'instance du repository en passant par l'injecteur 

5. Compiler et tester. 

> Que remarquez-vous ? Pourquoi ? 

## Ajouter des données dans la BDD
Afin d'éviter d'avoir une base de données vide, nous allons modifier la classe qui gère la base de données pour y insérer des données de tests à la création de la base de données. 

- Ajouter un callback lors de la l'instanciation de la BDD pour intercepter l'événement de création de la BDD. 
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
> Compilez et testez


## A vous de jouer
Maintenant vous allez coder les amis :) 

1. Modifier l'écran d'ajout pour sauvegarder les nouveaux voisins dans la base de données

2. Implémenter les actions de suppression, d'ajout en favoris et de like de voisins. 

> ATTENTION, vous n'avez pas besoin de raffraichir la liste, elle sera mise à jour automatiquement via le live Data. 


3. Ajouter un menu dans la toolbar permettant à l'utilisateur de choisir la source de données à utiliser : 
- Non persistent --> InMemory
- Persistent --> Local

4. Ajouter un menu dans la toolbar de la liste permettant de filtrer la liste en fonction des critères (tous, favoris, liked)

5. Ajouter une vue pour afficher le détail d'un voisin. 

5. Ajouter un bouton dans la vue de détail permettant d'ajouter un voisi en favori.
