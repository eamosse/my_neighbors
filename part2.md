## Partie 2: Interfaces d'ajout et de détails 
Dans cette section, nous allons ajouter deux nouveaux fragments permettant d'ajouter un nouveau voisin et voir les détails d'un voisin existant. 

### 1. Créer un nouveau fragment 

1. Ajouter un nouveau fragment ``AddNeighbourFragment`` dans le projet ainsi que la vue associée

2. Ajouter un layout pour dessiner la vue du fragment ``add_neighbor.xml``

> Vous allez devoir vous débrouller seuls, la vue doit ressembler à l'imge ci-dessous 

![Ajoutez un voisin](/add_neighbour.png "Nouveau voisin")

> Utilisez les composants MaterialComponents, qui permettent de dessiner des composants plus zolies

> Pour plus d'infos sur les composants MaterialComponents  
[voir le lien](https://material.io/develop/android/components)


```xml
<com.google.android.material.textfield.TextInputLayout
    android:id="@+id/addressLyt"
    style="@style/Widget.MaterialComponents.TextInputLayout.OutlinedBox"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:layout_marginTop="10dp">

    <com.google.android.material.textfield.TextInputEditText
        android:id="@+id/address"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:ems="10"
        android:hint="@string/hint_address"
        android:inputType="text" />
</com.google.android.material.textfield.TextInputLayout>

```
 
3. Associer le layout au fragment en utilisant le viewbinding

4. Gérer l'action sur le bouton ```enregistrer```. Quand l'utilisateur clique sur le bouton enregistrer :
    - Récupérer les valeurs de champs 
    - Créer un nouvel objet Neighbor 
    - Ajouter le nouveau voisin dans la liste

> Contraintes 

>> Valider les données des champs du formulaire 

    - Tous les champs sont obligatoires

    - L'email doit être valide (afficher une erreur sous le champ email indiquant à l'utilisateur qu'il y a une erreur)

    - Le bouton reste grisé tant que tous les champs ne sont pas validés
    
    - Le téléphone doit être valide (commencant par 06 ou 07 et doit avoir 10 charactères) -- (afficher une erreur sous le champ téléphone indiquant à l'utilisateur que le format doit être 0X XX XX XX XX XX)

    - Le champ bio A propos de moi doit autoriser au maximum 100 charactères 

    - Les champs Image et Website doivent être des URLs valides

>> Bonus, quand l'utilisateur rempli le champ Image avec un lien valide, afficher automatiquement un visuel de l'image à la place de l'image par défaut tout en haut. 

<details>
<summary>Besoin d'aide pour la gestion d'erreurs ou le nombre de charactères max ?</summary>
C'est par ici, https://material.io/develop/android/components/text-fields.
</details>

### ViewModel 

1. Créer un ViewModel ``ÀddNeighborViewModel````

2. Ajouter une méthode ```save``` dans le VM permettant de sauvegarder un nouveau voisin 

3. Associer le VM au fragment

### Gérer la navigation
Dans cette section, nous allons modifier l'activité ``MainActivity`` afin de faciliter la gestion du nouveau fragment.

1. Modifier la layout de l'activité principale pour y ajouter un bouton floatant (floating button)

```xml
    <com.google.android.material.floatingactionbutton.FloatingActionButton
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginEnd="15dp"
        android:layout_marginBottom="15dp"
        android:src="@drawable/ic_baseline_person_add_24"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent" />
```

2. Modifier l'actitivé pour intercepter le click sur le bouton flottant et lancer le fragment ``AddNeighborFragment``

- Tester

### Gestion de la toolbar 
Dans une application Android, une toolbar permet gérer la barre d'action. Une barre d'action contient le titre des écrans, ou les options de navigations. 

1. Modifier le layout de l'activité principal pour y ajouter une toolbar. 
> La toolbar doit se positionner tout en haut de l'écran et les autres vues doivent se positionner sous la toolbar. 

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <androidx.appcompat.widget.Toolbar
        android:id="@+id/toolbar"
        android:layout_width="match_parent"
        android:layout_height="?attr/actionBarSize"
        android:layout_weight="1"
        android:background="?attr/colorPrimary"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toRightOf="parent"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_scrollFlags="scroll|enterAlways"
        app:title="@string/app_name" />

    .....

</androidx.constraintlayout.widget.ConstraintLayout>

```

2. Modifier l'activité pour qu'elle utilise la toolbar pour gérer la navigation 

```kotlin
...
override fun onCreate(savedInstanceState: Bundle?) {
    ...
    setSupportActionBar(binding.toolbar)
    ...
}
...
```

3. Ajouter une interface ``NavigationListener`` dans le package principal du projet, elle sera utilisée pour faciliter la communication entre l'activité et les fragments.

```kotlin
    interface NavigationListener {
        fun changeFragment(fragment: Fragment)
    }
```

4. Ajouter une fonction ``setTitle`` dans l'interface ``NavigationListener`` permettant aux fragments de modifier le titre de la toolbar

```kotlin
interface NavigationListener {
    ...
    fun updateTitle(@StringRes title: Int)
}
```

5. Ajouter une fonction dans `NavigationListener` permettant de fermer un fragment 


6. Modifiez l'activité pour changer le titre de la toolbar à l'appel de la fonction ``updateTitle``

```kotlin
...
override fun updateTitle(title: Int) {
    binding.toolbar.setTitle(title)
}
...

```

7. Modifier les fragments ``ListNeighborsFragment`` et ``AddNeighborFragment`` pour afficher le bon titre dans la toolbar : 

- ListNeighborsFragment affiche ``Liste des voisins`` comme dans l'exemple

```kotlin
(activity as? NavigationListener)?.let { 
                it.updateTitle(R.string.fragment_list_title)
            }
```

 - AddNeighborFragment affiche ``Nouveau voisin``


### Ajouter une vue peremttant d'afficher les détails d'un voisin 

1. Créer le fragment et le view Model 

2. Faire passer un fragment selectionné de la liste des voisins à l'écran de détail

3. Adapter le titre de la toolbar 

4. Ajouter les actions (liker, favoris) dans la vue de détail 


## Partie 3: Gérer la persistance avec Room
[Base de données](part3.md)