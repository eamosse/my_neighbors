# Application   entre voisins

Dans ce TP, vous allez créer une application permettant de mettre en relation des voisins d'une localité. L'objectif est de créer un ensemble d'interfaces permettant: 
1. D'afficher la liste des voisins 
2. D'ajouter un voisin 
3. D'afficher les détails d'un voisin 
4. De supprimer un voisin

> Dans le cadre de ce TP, la liste des voisins sera gérée en mémoire vive, c'est-à-dire qu'une fois l'application fermée la liste sera réinitialisée. 

**Concepts couverts dans ce TP**
- Gestion de fragments 
- Utilisation de recyclerView et des adapteurs


## Partie 1: Afficher la liste des voisins

### Configuration du projet
Nous allons créer une application avec une seule activité. L'activité sera utilisée pour gérer la navigation et des fragments seront utilisés pour afficher les contenus de l'application.

1. Créer et configurer une nouvelle application

2. Modifiez le contenu du layout associé à l'activité 

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <androidx.fragment.app.FragmentContainerView
        android:id="@+id/fragment_container"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

</androidx.constraintlayout.widget.ConstraintLayout>
```


### Afficher la liste des voisins dans un fragment 

1. Créer un nouveau package `ui`. Dans le nouveau package créez un sous-package `fragments`

2. Daplacer l'activité dans le package `ui`

3. Dans le package `fragments`, créer une classe `ListNeighborsFragment`

4. Modifier le fragment pour qu'il étende de la super-classe `Fragment`

```kotlin 
    class ListNeighborsFragment : Fragment(){
}
```

### Mise en place du layout associé au fragment

1. Créer un layout `neighbors_list_fragment`

2. Modifier le layout pour y ajouter un recyclerview

3. Ajoutez un autre layout **neighbor_item** qui represente la manière dont chaque Neighbor sera affiché dans la liste

> On souhaite afficher la photo du voisin, son nom, une partie de sa bio, un lien vers son profile facebook, une icone permettant de l'ajouter en favori, une icone pour le liker et une icone pour le supprimer. 

4. Modifier le fragment `ListNeighborsFragment` pour l'associer au layout `neighbors_list_fragment`

```kotlin
    /**
     * Fonction permettant de définir une vue à attacher à un fragment 
     */
    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View? {
        TODO("Utiliser du viewbinding")
        return inflater.inflate(R.layout.list_neighbors_fragment, container, false)
    }

```

### Modifiez l'activité **MainActivity** pour afficher le fragment 

1. Ajouter une méthode dans l'activité permettant de changer de fragment 

```kotlin 
 private fun changeFragment(fragment: Fragment) {
        supportFragmentManager.beginTransaction().apply {
            replace(R.id.fragment_container, fragment)
            addToBackStack(null)
        }.commit()
    }

```

2. Adapter l'activité pour qu'au lancement elle affiche le fragment `ListNeighborsFragment`

### Gestion de données 

1. Ajouter un nouveau package ```dal``` puis un sous-package ```models```

2. Créer une classe permettant de modéliser les voisins

``` kotlin
data class Neighbor(
    val id: Long,
    val name: String,
    val avatarUrl: String,
    val address: String,
    val phoneNumber: String,
    val aboutMe: String,
    val favorite: Boolean,
    val webSite: String
)
```

4. Ajouter deux nouveaux packages `repositories` et `services`

6. Dans le package `services`, ajouter une interface `NeighborService` qui définit les principales opérations qu'on peut effectuer sur les voisins. 

> On veut pouvoir : i) lister les voisins, ii) ajouter un voisin, iii) supprimer un voisin, iv) ajouter un voisin en favori et v) liker un voisin

7. Implémenter un service `InMemoryNeighbor` conformement à l'interface `NeighborService` et qui gère les données en mémoire. 

> Voir le fichier [Exemple de données](in_memory.md) pour une liste de voisins de tests

5. Dans le package `repositories`, ajouter une classe `NeighborRepository` qui expose les opérations du service `NeighborService`

> Le repository ne doit faire aucun traitement, les traitements sont réalisés par le service; le repository joue le rôle d'orchestrateur.

6. Transformer le repository en singleton

- Modifiez la classe NeighborRepository
```kotlin 
class NeighborRepository private constructor() {
    private val apiService: NeighborApiService

    init {
        apiService = InMemoryNeighbor()
    }

    companion object {

        @Volatile private var instance: NeighborRepository? = null

        fun getInstance() =
            instance ?: synchronized(this) { 
                instance ?: NeighborRepository().also { instance = it }
            }
    }
}
````

## Afficher les neighbors dans la liste 
1. Ajouter un adapter, permettant de gérer l'affichage des données de la liste 

2. Configurer le recyclerview en lui associant une instance de l'adapter

> Il est recommandé de configurer les vues d'un fragment après la création de la vue principale de celui-ci. Pour cela, il faudra configurer le recyclerview dans la méthode `onViewCreated` du fragment

```kotlin 
override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
    super.onViewCreated(view, savedInstanceState)
    val neighbors = NeighborRepository.getInstance().getNeighbours()
    val adapter = ListNeighborsAdapter(neighbors)
    recyclerView.adapter = adapter
}
```

3. Executez le projet 

> A ce stade, la liste des voisins doit s'afficher

### Mise en place de l'architecture MVVM
Notre code fonctionne mais ne respecte pas les bonnes pratiques d'une architecture clean, c'est à dire une bonne séparation entre la logique métier et la vue.

Dans les bonnes pratiques, la vue doit utiliser un viewmodel pour gérer le traitement de ses données. 

1. Dans le package `fragments`, ajouter une classe `ListNeighborViewModel` qui hérite de la classe `ViewModel`. 

2. Modifier le VM en y ajoutant une instance du repository comme attribut

```kotlin
private val userRepository: UserRepository = UserRepository.getInstance()
```

3. Ajouter un LiveData qui permet au VM de notifier la vue quand les données changent

```kotlin
private val _neighbors: MutableLiveData<List<Neighbor>> = MutableLiveData()
val neighbors: LiveData<List<Neighbor>> = _neighbors
```

4. Adapter le VM pour charger la liste des voisins à la création d'une instance

5. Modifier le fragment `ListNeighborsFragment` pour instancier le VM et observer les changements sur la liste des voisins

```kotlin
class ListNeighborsFragment: Fragment() {
    private val viewModel: ListNeighborViewModel by lazy {
        ViewModelProvider(this)[ListNeighborViewModel::class.java]
    } 

    ...

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        ...
        viewModel.neighbors.observe(viewLifeCycleOwner) {
            adapter.submitList(it)
        }
}

}
```

> A partir de maintenant, tous les traitements de données se font dans le VM.

### Gestion des actions utilisateurs
Pour faciliter la communication entre le fragment et l'adateur, on utilise une interface qui est implémentée dans le fragment et appelée dans l'adapteur. 

1. Ajouter un package ``adapters`` puis une classe ``ListNeighborHandler``

```kotlin
    interface ListNeighborHandler {
        fun onDeleteNeibor(neighbor: Neighbor)
    }
```

2. Modifier le fragment ``ListNeighborsFragment`` pour implémenter l'interface ``ListNeighborHandler``

3. Modifier le constructeur de l'adapteur pour y ajouter un constructeur par défaut penant en paramètre une instance de l'interface ``ListNeighborHandler``


4. Modifiez la fonction ``onBindViewHolder`` dans ``ListNeighborHandler`` pour intercepter le clique sur le bouton ``delete``. 

5. Modifiez la fonction ``onDeleteNeibor`` dans le fragment pour afficher une dialogue quand l'utilisateur clique sur le bouton supprimer. 

6. Ajouter les différentes méthodes pour supprimer un voisin


## Implémenter les autres cas d'utilisation

1. Ajouter un voisin en favori

2. Liker un voisin 

3. Bonus : Comme Tinder :) Swiple left pour liker, swipe right pour supprimer 

## Partie 2: Interfaces d'ajout et de détail d'un voisin
[Ajoutez un voisin](part2.md)
