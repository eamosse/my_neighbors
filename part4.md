## Partie 4: Synchroniser les données avec un API REST 

# Description de l'API
L'url de base de l'API étant : https://dmi1.glitch.me

### L'API supporte les actions suivantes 
1. Créer un voisin `POST /neighbors`
2. Lister les voisins `GET /neighbors`
3. Like un voisin `PUT /neighbors/{id}/like`
4. UnLike un voisin `PUT /neighbors/{id}/unlike`
5. Ajouter en favori `PUT /neighbors/{id}/star`
6. Supprimer des favoris `PUT /neighbors/{id}/unstar`
5. Supprimer un voisin `DELETE /neighbors/{id}`

### Modèle de réponse des actions de l'API
Toutes les actions de l'API retournent une structure JSON contenant 3 clés : 
- status : OK (L'action a tété exécutée correctement) ou KO (une erreur s'est produite)
- message : Un text décrivant le status; dans le cas d'un status KO le text correspond au message d'erreur
- data : Réponse de l'action, la structure de cette clé varie en fonction de l'action. Dans le cas d'une demande de liste c'est un JSONARRAY avec la liste des voisins. 

## Structure d'un object voisin  
```json
{
  "_id": "65d7ab91441e3615d7832486",
  "name": "John Doe",
  "avatarUrl": "https://example.com/avatar.jpg",
  "address": "123 Main Street, Cityville",
  "phoneNumber": "+1 555-1234",
  "aboutMe": "I love programming and exploring new technologies.",
  "favorite": true,
  "like": false,
  "webSite": "https://johndoe.com"
}
```
> Vous pouvez utiliser Postman pour explorer l'API avant de l'intégrer dans vos projets 

# Synchroniser l'app avec l'API 
Dans cette section, vous allez implémenter une nouvelle source de données ``OnlineNeighbor``. 

1. Ajouter les dépendances à Retrofit dans le fichier gradle de l'app

```kotlin
// Retrofit
implementation("com.squareup.retrofit2:retrofit:2.9.0")
// Retrofit with Scalar Converter
implementation 'com.squareup.retrofit2:converter-moshi:2.9.0'
```

2. Dans `dal`, créer un nouveau package `online`

3. Dans `online`, créer un package `response` et dans ce package ajouter une nouvelle classe qui permettra de désérialiser les réponses de l'API 
Pour la liste des voisins on attend un objet json semblable à l'exemple ci-dessous. 
<details>
<summary>Voir l'exemple</summary>

```json
{
    "data": [
        {
            "_id": "65d7ab91441e3615d7832486",
            "id": 123,
            "name": "John Doe",
            "avatarUrl": "https://example.com/avatar.jpg",
            "address": "123 Main Street, Cityville",
            "phoneNumber": "+1 555-1234",
            "aboutMe": "I love gardening and reading books.",
            "favorite": true,
            "webSite": "https://johndoe.com",
            "created_at": "2024-02-22 20:16:17",
            "last_updated": "2024-02-22 20:16:17",
            "like": true
        }
    ],
    "message": "Liste des voisins",
    "status": "OK"
}
```
</details>

Pour désérialiser cet objet json en Kotli, vous aurez besoin d'une classe contenant les propriétés suivantes : 
- message : String 
- status : String
- data : List<Neighbor>

... et suivant le même principe, il faudra aussi créer une classe `Neighbor`. 

> Cette classe `Neighbor` est différente des deux classes existantes de l'application, elle est propre à la couche d'accès API tout comme l'entité Neighbor est propre à la base de données Room. 

> Pour simplifier la création d'objets liés aux web services, il existe des plugins Android Studio comme Json To Kotlin Class. Ces librairies permettent de générer les classes de désérialiser à partir d'un object JSON correspondant à la réponse d'un web service. 

```kotlin
package org.mbds.unice.github.ui.users

data class NeighborsResponse(
    val `data`: List<Data> = emptyList(),
    val message: String? = null,
    val status: String? = null
)

data class Data(
    val _id: String? = null,
    val aboutMe: String? = null,
    val address: String? = null,
    val avatarUrl: String? = null,
    val created_at: String? = null,
    val favorite: Boolean = false,
    val last_updated: String? = null,
    val like: Boolean = false,
    val name: String? = null,
    val phoneNumber: String? = null,
    val webSite: String? = null
)
```

> Il est recommandé d'utiliser des attributs nullables dans les classes de web service. En effet, la conversion JSON -> Objet plante si une clé n'existe pas dans le JSON.   

4. Dans `online`, ajoutez une interface `NeighborService`

```kotlin
import retrofit2.http.GET

interface NeighborService {
    @GET("neighbors")
    suspend fun getNeighbors() : NeighborsListResponse
}

```
<details>
<summary>Explications</summary>

- Retrofit utilise les interfaces pour déterminer les appels API à réaliser. L'interface contient, l'ensemble des endpoints qu'on veut appeler dans l'API ainsi que leurs paramètres d'entrées, données de retour et le verbe HTTP utilisé. Il n'est pas nécessaire d'implémenter cette interface, c'est Retrofit qui le fait, comme les DAO dans Room.

- L'annotation @GET indique à Retrofit qu'il faut faire une requête HTTP de type GET

- Le mot clé suspend permet d'indiquer à Kotlin que cette fonction effectue un appel bloquant; il force le developpeur à l'appeler dans une coroutine et par conséquent éviter de bloquer le thread principal.

- Finalement, le type de retour de la fonction correspond à l'objet qu'on attend en retour. 

</details>

5. Configurer Retrofit 

- Dans le package `online`, créez une classe `OnlineNeighbor` qui implémente l'interface `NeighborService`. Implementez toutes les méthodes de cette interface comme pour les deux sources de données du TP précédent. 

- Ajouter une variable `BASE_URL` dans `NeighborService` qui contient l'url de base du Web Service. 

> Cette variable doit être une constante et statique

> Connaissez-vous la différence entre une constante et une variable déclarée avec le mot clé `val` ? 

- Dans `NeighborService`, ajouter les lignes suivantes 

```kotlin
private val retrofit = Retrofit.Builder()
    .baseUrl(BASE_URL)
   .addConverterFactory(MoshiConverterFactory.create())
   .build()

private val retrofitService : NeighborService by lazy {
       retrofit.create(NeighborService::class.java)
    }
```
<details>
<summary>Explications </summary>

1) La variable retrofit est utiliséé pour instancier un objet retrofit qui sera utilisée pour faire les requêtes HTTP. Retrofit a besoin de deux configurations minimum : 
- l'url de base de l'API (la partie commune à toutes les requêts)
- un converter : Bibliothèqye qui sera utilisée pour transformer les objets POJO en JSON et vice-verca.

2) La variable retrofit service est utilisé pour indique à retrofit l'interface contenant les actions de l'API à implémenter.

</details>

6. Implémenter la méthode `getNeighbors`
Nous pouvez maintenant faire un appel à l'API en utilsant Retrofit. 

- Modifier la méthode `getNeighbors` pour faire un appel au web service, récupérer la réponse du service, transformer la réponse au bon format et mettre à jour le LiveData. 

```kotlin
fun getNeighbors() : LiveData<List<Neighbor>> {
    val _neighbors = MutableLiveData(List<Neighbor>)
    CoroutineScope(Dispatchers.IO).launch {
        val response = retrofitService.getNeighbors()
        if (response.status == "OK") {
            val result = response.data.map {
                it.toNeighbor()
            }

            _neighbors.postValue(result)
        }
    }
    return _neighbors
}

```

<details>
<summary>Explications </summary>
- On crée une coroutine en précisant le context Dispatchers.IO, on veut faire l'appel au web service sur un thread secondaire. 

- On utilise le service pour appeler l'API, Retrofit s'occupe de générer la requête HTTP, l'éxécuter et convertir la réponse en objet Kotlin

- On vérifie que le statut est OK

- On convertit chaque voisin retourné par le web service en objet métier (ie. le même type que ceux utilisés par la vue)

- Finalement on met à retourne un liveData, qui sera mis à jour quand les données du web service sont disponibles. 

</details>


### Configurer le repository pour utiliser la source de données online 

- Modifier le repository pour prendre en compte la nouvelle source de données. 


### Finaliser le travail 
- Implémentez toutes les méthodes de la source de données online en faisant appel à la bonne action de l'API

- Modifier le menu pour pouvoir chosir entre (memory, local, online)

- Faire en sorte de basculer automatiquement sur le mode local si le téléphone n'a pas accès à internet 

### BONUS +++
- Mettre en place un mécanisme qui permet de synchroniser les données sauvegardées en locale quand le téléphone récupère l'accès à internet. 