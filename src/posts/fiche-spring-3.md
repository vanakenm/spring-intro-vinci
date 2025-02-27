---
title: Fiche 3
description: Spring et sécurité.
permalink: posts/{{ title | slug }}/index.html
date: '2025-01-30'
tags: [fiche, spring]
---

# Sécurisation

La semaine passée nous avons appris les concepts suivants:

- Persistence des données
- Object-Relationnal Mapping

Cette semaine nous allons ajouter les derniers éléments important pour tout backend : la gestion de la sécurité et de l'authentification.

## Objectif du projet

Nous allons partir d'une API pour la gestion d'une pizzeria, dont les fonctionnalités suivantes ont été déjà développées :

- Lister les pizzas disponibles (une pizza a un nom et un contenu)
- Recupérer les informations d'une pizza à partir de son identifiant
- Créer une nouvelle pizza
- Supprimer une pizza
- Editer une pizza
- Commander une pizza
- Inscrire des utilisateurs (un utilisateur a un pseudonyme, un mot de passe en clair pour le moment et un rôle utilisateur par défaut)

L'objectif de ce tutoriel va être de protéger les actions qui doivent l'être. 

- Seuls les utilisateurs connectés pourront commander une pizza
- Seuls les administrateurs pourront créer, supprimer et éditer des pizzas.

On va donc couvrir ici deux notions différentes et liées:

- On parle d'`Authentification` pour tout ce qui est "est ce que la personne est bien qui elle prétend être". La manière la plus classique de gérer de l'authentification est via un user (souvent un email) et un password.
- On parle d'`Authorization` pour tout ce qui est "est ce que cette personne a le droit de faire ce qu'elle souhaite" - un user peut être correctement authentifié mais ne pas avoir les droits de faire une certaine opération (créer une nouvelle pizza dans notre exemple). La solution classique s'appelle des `Roles` ou [RBAC](https://en.wikipedia.org/wiki/Role-based_access_control) - un système par lequel chaque user se voit attribuer un ou plusieurs rôles qui lui donne des droits spécifique (user, administrateur ou par exemple pour un site de contenu: reader, editor, reviewer, ...)

On voit directement qu'on ne peut pas `authoriser` quelqu'un avant de l'avoir `authentifié`.

## Project setup

Nous allons cloner un repository contenant le code déjà existent pour le modifier dans cette troisième fiche.

Cloner le repository présent ici: [https://github.com/e-vinci/cae_exercices_fiche3](https://github.com/e-vinci/cae_exercices_fiche3).

Lancez le projet (avec le docker pour Postgres ou non selon vos préférence) et vérifiez que tout est en ordre.
Notez que le postgres fourni est fait pour tourner sur le port 5433 (et non sur le 5432 par défaut) dans le but de ne pas créer de conflit avec un Postgres installé sur votre machine.

### DTOs

Vous allez trouvez deux répertoire au niveau du package "models":
- Un "entities" qui correspond aux objets liés à la DB (les "Entités") comme précédemment
- Un nommé "DTOs" avec de "simples objets java" ("POJO" en anglais: "Plain Old Java Object")

Ceci suit un pattern architectural appellé ["Data Transfer Object"](https://martinfowler.com/eaaCatalog/dataTransferObject.html) - le site référencé est celui de Martin Fowler, un programmeur et architecte assez connu et prolifique notemment dans sa documentation de "patterns" - des éléments de code que l'on retrouve dans beaucoup d'applications. 

L'idée ici est d'éviter que les objet de type "Entity" ne soient utilisé en input du controller - il n'y a par exemple aucun sens que l'utilisateur fournissent l'id de l'objet, ou le password encrypté, bien que ces champs doivent exister (et être obligatoires) au niveau de l'entité User. On crée donc des objets distincts, très simples (des champs et des accesseurs) pour ce faire.

## Tutoriel

Le tour du projet étant fait et fonctionnel (le serveur tourne et http://localhost:8080/pizzas renvoie une liste - potentiellement vide), nous allons ajouter ces features de sécurité.

### Nouvelles dépendances

Nous allons ajouter des dépendances nécessaires:

- Dans le pom.xml, faire "Edit Starters" (il faut avoir le pom.xml ouvert, l'option est proposée dans l'éditeur même) et ajouter spring security (Edit Starter permet de compléter vos settings après la création du projet)
- Editer le pom.xml pour ajouter java-jwt (celui de com.auth0) et spring-dotenv - pour ce faire, utilisez "Generate... > Dependency"

![starters](https://raw.githubusercontent.com/e-vinci/cae-exercices-syllabus/4550b38cd10a467862fef524540efc1714280126/src/posts/edit-starteds.png)

Faite attention de prendre les packages exacts spécifiés (il y en a beaucoup, les confusions sont faciles !).

Pourquoi ces packages:

- **Spring Security** va nous permettre de facilement configurer les accès à nos différent endpoints
- **JWT (JSON Web Token)** est une technologie pour générer des "token" après une authentification réussie. Les token servent d'alternatives aux cookies dès lors que l'on utilise un client type SPA (comme React) - des éléments sur ce sujet [ici](https://stackoverflow.com/questions/37582444/jwt-vs-cookies-for-token-based-authentication)
- **Spring Dotenv** va nous aider à garder nos secrets (typiquement user/password utilisé pour la database) dans un format plus facile à manipuler que des variables d'environnement.

Une fois ceci fait, relancez l'application et retournez sur http://localhost:8080/pizzas - que voit on?

Bien que nous n'ayons fait aucune configuration, l'application n'est plus accessible sans login/password - ceci est du au fait que Spring Security fonctionne sur un principe de "deny all". En d'autres mots - si on ne spécifie rien, toutes les routes nécessitent que le user soit authentifié. Cette valeur par défaut nous embête un peu ici... mais elle a beaucoup de sens !

Nous allons voir dans la suite comment mettre à jour cette configuration.

### Hachage des mots de passe

En première étape, on va encrypter les mots de passe avant de les sauvegarder dans la DB. L'idée est qu'on ne devrait jamais stocker de mots de passe "en clair" (ie sans êtres encryptés). N'oubliez pas que si quelqu'un met la main sur les mots de passes utilisateurs de votre application il peut non seulement l'utiliser en se faisant passer pour un de ces utilisateurs... mais également tenter sa chance avec ce même email et mot de passe dans d'autres applications (la plupart des gens utilisent les mêmes mots de passe).

Il existe un grand nombre de manière d'encrypter un mot de passe - celle que l'on va utiliser (la plus courante, basée sur [bcrypt](https://en.wikipedia.org/wiki/Bcrypt)) est une opération à sens unique. En d'autre mot, si quelqu'un met la main sur le mot de passe encrypté *il lui est impossible de retrouver le mot de passe d'origine*.

Mais alors, comment faire pour vérifier après ? La logique est en fait assez simple:

- Le user créer son user et son mot de passe (en fournissant le mot de passe en clair)
- On l'encrypte avec bcrypt et on sauve cette version dans la base de données
- Quand le user revient et veut se connecter, il tape son mot de passe (à nouveau en clair)
- On l'encrypte, et on *compare la version encryptée avec celle dans la base de données* 

L'algorythme bcrypt a été implémenté dans un grand nombre de language dont en Java. La bibliotèque qui le contient est intégrée à Spring Security. On appelle cette opération d'encryption un "hashage".

On peut faire une première version très simple en mettant à jour le UserService:

```java
    public void createOne(String username, String password) {
        BCryptPasswordEncoder encoder = new BCryptPasswordEncoder();

        User user = new User();
        user.setUsername(username);

        String hashedPassword = encoder.encode(password);
        user.setPassword(hashedPassword);
        user.setRole("USER");
        userRepository.save(user);
    }
```

La classe BCryptPasswordEncoder est présente dans le package de spring, et dispose d'une méthode "encode" que l'on utilise sur le password avant de sauvegarder.
Pour tester ceci, le plus simple au stade actuel est d'utiliser un runner dans l'application (comme précédemment pour créer des données):

```java
    @Bean
    public CommandLineRunner demo(UserService userService) {
        return (args) -> {
            System.out.println("Creating users");
            userService.register("admin", "admin");
            userService.register("user", "user");
        };
    }
```

Une fois l'application relancée, allez voir la DB via DataGrip - vous devriez voir deux record avec les passwords correctement encryptés:

| id | password                                        | username | role |
|----|------------------------------------------------|----------|------|
| 2  | $2a$10$BGc4auhIrzMrEdsX3Xq25u9zhwY1Zl.emrSP2dTwUBYEyVOxGRBkC | admin    | USER |
| 3  | $2a$10$.Dt44U9Le.w3QNTyAhTQ/ua2z7xCuqbDbu.b5A6Vgm8DFnnp1yYOW | user     | USER |


Ceci dit, la manière dont on utilise le BCryptPasswordEncoder n'est pas très efficiente: on va recréer une instance à chaque usage. On peut adapter ceci pour utiliser le système d'injection de dépendance de Spring - on ne peut pas l'injecter directement (elle n'a pas d'annotation type @Service), nous allons donc procéder autrement.

Créer un package "configuration" et dedans une classe BcryptConfiguration

```java
@Configuration
public class BcryptConfiguration {

    @Bean
    public BCryptPasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

}
```

[@Configuration](https://docs.spring.io/spring-framework/reference/core/beans/java/configuration-annotation.html
) indique a Spring que cette classe n'est pas elle même un Bean, mais que ses méthodes peuvent en créer - ici la méthode "passwordEncoder" retourne un Bean de type BCryptPasswordEncoder... qui est donc lui même injectable.

On peut donc l'injecter dans notre UserService - et l'utiliser en lieu et place de celle créee sur le moment:

```java
public class UserService {

    private final UserRepository userRepository;
    private final BCryptPasswordEncoder passwordEncoder;

    public UserService(UserRepository userRepository, BCryptPasswordEncoder passwordEncoder) {
        this.userRepository = userRepository;
        this.passwordEncoder = passwordEncoder;
    }

    ...

      public void createOne(String username, String password) {
        User user = new User();
        user.setUsername(username);

        String hashedPassword = passwordEncoder.encode(password);
        user.setPassword(hashedPassword);
        user.setRole("USER");
        userRepository.save(user);
    }
}
```

## Création de token JWT

Nous avons maintenant des users avec des passwords correctement "hashés" dans notre base de donnée et un bean BCryptPasswordEncoder capable de vérifier si un password fourni (en clair) est identique à celui dans la DB (hashé).

Reste que dans notre architecture, il va nous falloir renvoyer autre chose qu'un cookie si le user fourni (correctement) son user et mot de passe - à savoir un "token" JWT.
Le principe est que sur un login réussi, on va renvoyer le user et un token permettant de s'authentifier lors des requêtes suivantes (vous avez sans doute déjà fait cela côté client dans vos cours de React).

Comme vu plus haut dans "DTO", on ne veut pas renvoyer l'objet user complet vers le client, mais juste le username et le token. On va donc créer un DTO dédié pour ceci:

```java
@Data
@NoArgsConstructor
public class AuthenticatedUser {
    private String username;
    private String token;
}
```

Remarquer le nommage - il s'agit bien d'un utilisateur authentifié.

Pour créer un token JWT il nous faut trois choses:

- Un algorithme
- Un secret
- Une durée de vie

Allons rajouter cela dans le UserService:

```java
public class UserService {
    ...
    private static final String jwtSecret = "ilovemypizza!";
    private static final long lifetimeJwt = 24*60*60*1000; // 24 hours

    private static final Algorithm algorithm = Algorithm.HMAC256(jwtSecret);
```

Notre secret n'est pas réellement très secret, nous changerons cela dans un moment. L'algorithme choisi est [HMAC](https://en.wikipedia.org/wiki/HMAC)-256 - le 256 indique la "force" de la fonction (et donc la difficulté de la casser).

La durée parait raisonnable - c'est un équilibre entre sécurité et "convenience" - on ne veut pas demander au user de se reconnecter toutes les cinq minutes, mais sans doute plus d'une fois par an - en fonction de l'importance des données qui sont derrière.


Ceci nous permet de créer le token correctement:

```java
    public AuthenticatedUser createJwtToken(String username) {
        String token = JWT.create()
                .withIssuer("auth0")
                .withClaim("username", username)
                .withIssuedAt(new Date())
                .withExpiresAt(new Date(System.currentTimeMillis() + lifetimeJwt))
                .sign(algorithm);

        AuthenticatedUser authenticatedUser = new AuthenticatedUser();
        authenticatedUser.setUsername(username);
        authenticatedUser.setToken(token);

        return authenticatedUser;
    }
```

On voit ici que le token ne se base en rien sur le mot de passe - mais juste sur le username. On doit donc bien vérifier *avant* que le password est correct, puis seulement appeller la méthode pour créer le token.

On a maintenant toutes les pièces pour pouvoir logger correctement un utilisateur, reste à les mettre ensemble dans une méthode login dans le UserService

```java
    public AuthenticatedUser login(String username, String password) {
        User user = userRepository.findByUsername(username);
        if (user == null) return null;
        if (!passwordEncoder.matches(password, user.getPassword())) {
            return null;
        }
        return createJwtToken(username);
    }
```

(rajoutez la méthode `findByUsername` si elle n'existe pas encore sur le UserService)

Celle ci fait donc trois opérations:

- Récupère un user dans la base de données basé sur le username
- Vérifie (via le password encoder) si le mot de passe est correct
- Si oui, créer le token et le renvoie

On appelle pas directement un service, donc il faut également avoir la méthode dans le controller:

```java
    @PostMapping("/login")
    public AuthenticatedUser login(@RequestBody Credentials credentials) {
        System.out.println("Loggin in controller");
        System.out.println(credentials);
        if (credentials == null ||
                credentials.getUsername() == null ||
                credentials.getUsername().isBlank() ||
                credentials.getPassword() == null ||
                credentials.getPassword().isBlank()) {
            throw new ResponseStatusException(HttpStatus.BAD_REQUEST);
        }

        AuthenticatedUser user = userService.login(credentials.getUsername(), credentials.getPassword());
        if (user == null) throw new ResponseStatusException(HttpStatus.UNAUTHORIZED);

        return user;
    }
```

Il nous faut encore pouvoir vérifier que le token est correct - ajoutons une méthode de plus au UserService:


```java
    public String verifyJwtToken(String token) {
        try {
            return JWT.require(algorithm).build().verify(token).getClaim("username").asString();
        } catch (Exception e) {
            return null;
        }
    }
```

La méthode délègue cette opération au package JWT, sur base du token fourni, de la variable qui a permis de le créer et du secret (via l'algorithme).

Reste à s'assurer que toutes les (autres) méthodes vérifient bien le token à chaque appel 

### Création d'un filtre de vérification

Spring Security nous permet de définir des "filtres" - des classes qui vont agir à chaque requête et les modifier voir les bloquer si certaines conditions ne sont pas remplies.

Nous allons créer un JWTAuthentificationFilter qui va vérifier si le token est correct à chaque requête:


```java
@Configuration
public class JWTAuthentificationFilter  extends OncePerRequestFilter {

    private final UserService userService;

    public JWTAuthentificationFilter(UserService userService) {
        this.userService = userService;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        String token = request.getHeader("Authorization");
        if (token != null) {
            userService.verifyJwtToken(token);
        }
        filterChain.doFilter(request, response);
    }
}
```

Le code en soit n'est pas bien complexe:

- La méthode "doFilterInternal" est appellée par Spring à chaque requête (on va voir juste après comment)
- Elle récupère le header "Authorization", et appelle le service pour vérifier le token
- On ne fait rien d'autre - parce que le service va lever une exception si le token n'est pas correct (et donc arrêter la requête)

### Configuration de la sécurité

Reste à lier le tout avec un fichier de configuration:

```java  
@Configuration
@EnableWebSecurity
@EnableMethodSecurity()
public class SecurityConfiguration {

    private final JWTAuthentificationFilter jwtAuthenticationFilter;

    public SecurityConfiguration(JWTAuthentificationFilter jwtAuthenticationFilter) {
        this.jwtAuthenticationFilter = jwtAuthenticationFilter;
    }

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
                .csrf(AbstractHttpConfigurer::disable)
                .sessionManagement(sessionManagement ->
                        sessionManagement.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
                .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class)
                .build();
    }
}
```

Du côté annotation, en plus du @Configuration on trouve:

- @EnableWebSecurity
- @EnableMethodSecurity()

qui vont nous permettre de définir pour chaque méthode les droits nécessaires.

La classe récupère notre filtre (via injection), et configure toute requête HTTP avec les opérations suivantes:

- Désactiver les [CSRF](https://en.wikipedia.org/wiki/Cross-site_request_forgery) - "Cross Site Request Forgery" - un mécanisme permettant de bloquer les requêtes venant d'un domaine différent. Quelque chose de logique dans une application backend, mais qui bloquerait toutes les requêtes d'une application type "SPA" (vu que le client tourne sur une autre url)
- Indiquer que les sessions sont STATELESS (sans état, le mode par défaut)
- Indiquer que notre filtre doit être appellé avant celui standard de Spring

### Test!

Il est plus que temps de tester tout ceci - un petit fichier .http devrait faire l'affaire:

{% raw %}
```bash
@baseUrl = http://localhost:8080

### Try to login an unknow user
POST {{baseUrl}}/auths/login
Content-Type: application/json

{
  "username":"unknown",
  "password":"admin"
}

### Login the default admin
POST {{baseUrl}}/auths/login
Content-Type: application/json

{
  "username":"admin",
  "password":"admin"
}


### Create the manager user
POST {{baseUrl}}/auths/register
Content-Type: application/json

{
  "username":"manager",
  "password":"manager"
}

### Login the manager user
POST {{baseUrl}}/auths/login
Content-Type: application/json

{
  "username":"manager",
  "password":"manager"
}
```
{% endraw %}

Les deux premiers cas devraient démontrer l'application - le user "inconnu" recoit une erreur 401, tandis que l'admin recoit bien en retour un json avec un token.

L'avant dernière créer un nouvel utilisateur puis se logge avec.

Notre volet "authentification" est fonctionnel - reste que tout le monde peut toujours tout faire !

### Authorisation et rôles

Pour faire la différence entre nos deux utilisateurs nous allons donner un rôle à l'administrateur. Utilisez DataGrip ou IntelliJ pour modifier l'utilisateur `admin` directement dans la base de données, pour lui donner le rôle `ADMIN`.

Une fois ceci fait il est possible de lire ces rôles et de donner des droits en conséquences. La bonne place pour faire ce code est évidemment le filter:


```java
@Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        String token = request.getHeader("Authorization");
        if (token != null) {
            String username = userService.verifyJwtToken(token);
            if (username != null) {
                User user = userService.readOneFromUsername(username);
                if (user != null) {
                    List<GrantedAuthority> authorities = new ArrayList<>();
                    if (user.getRole().equals("ADMIN")) authorities.add(new SimpleGrantedAuthority("ROLE_ADMIN"));
                    UsernamePasswordAuthenticationToken authentication = new UsernamePasswordAuthenticationToken(user, null, authorities);
                    authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                    SecurityContextHolder.getContext().setAuthentication(authentication);
                }
            }
        }
        filterChain.doFilter(request, response);
    }
```

> Nous avons besoin de pouvoir récupérerr un utilisateur à partir de son username (méthode "readOneFromUsername") - celle ci n'existe pas encore sur le service, à vous de la créer (on va éviter d'aller directement dans le repository à partir du filtre).

On a donc le début du filtre comme précédemment, mais on ne se contente plus de vérifier que le token est correct - en fonction du rôle de l'utilisateur on va lui fournir des droits ("GrantedAuthority"). Il faut bien voir ici la découpe entre notre code et celui de Spring:

- A nous de définir où un utilisateur est stocké (pour nous dans une table User d'une base de donnée Postgresql) et de quel type il s'agit (on regarde le champs rôle)
- A Spring de gérer ces "authorities" quand l'utilisateur se connecte.

En d'autre mot il y a une indirection entre à quoi servent les droits et d'où ils sont issus.

Ceci ne devrait rien changer à l'exécution du ficher .http - mais on peut maintenant sécuriser nos différentes méthodes via de simples annotations:

```java
@RestController
@RequestMapping("/pizzas")
public class PizzaController {
    ...
    @PostMapping({"", "/"})
    @PreAuthorize("hasRole('ROLE_ADMIN')")
    public Pizza addPizza(@RequestBody NewPizza newPizza) {
```

L'unique ajout et le "PreAuthorize" qui indique à Spring de vérifier que l'utilisateur courant a bien le rôle "ROLE_ADMIN" - sans quoi il ne peut pas effectuer l'opération.

Ceci est de la sécurité *déclarative* - pas besoin de mettre des "if" partout dans le code, on peut garder cette gestion à un seul endroit (le controller - pourquoi là ?).

### Test de sécurité

Voici à nouveau un .http pour vérifier tout ceci:

{% raw %}
```bash
######### NORMAL OPERATION  ###########

### Read all pizzas
GET http://localhost:3000/pizzas

### Read all pizzas with File variable
@baseUrl = http://localhost:8080

GET {{baseUrl}}/pizzas

### Create a pizza by using the admin account
#### First login as the admin
POST {{baseUrl}}/auths/login
Content-Type: application/json

{
  "username":"admin",
  "password":"admin"
}

> {% client.global.set("adminToken", response.body.token) %}

#### Create a pizza with the admin token
POST {{baseUrl}}/pizzas
Content-Type: application/json
Authorization: {{adminToken}}

{
  "title":"Magic Green",
  "content":"Epinards, Brocolis, Olives vertes, Basilic"
}

######### ERROR OPERATION  ###########

### 1. Create a pizza without a token
POST {{baseUrl}}/pizzas
Content-Type: application/json

{
  "title":"Magic Green",
  "content":"Epinards, Brocolis, Olives vertes, Basilic"
}

### 2. Create a pizza without being the admin, use manager account
#### 2.1 First login as the manager
POST {{baseUrl}}/auths/login
Content-Type: application/json

{
  "username":"manager",
  "password":"manager"
}

> {% client.global.set("managerToken", response.body.token) %}

#### 2.2 Try to create a pizza with the manager token
POST {{baseUrl}}/pizzas
Content-Type: application/json
Authorization: {{managerToken}}

{
  "title":"Magic Green",
  "content":"Epinards, Brocolis, Olives vertes, Basilic"
}
```
{% endraw %}

Soit:

- Si on se log en temps qu'admin et que l'on fourni bien le token, créer une pizza fonctionne
- Tous les autres cas (pas de token, token user) échouent.


## Sécurisation de properties

Nous parlons de sécurité... mais on vient en réalité d'introduire un gros problème à ce niveau - nos informations "sensibles" sont simplement commitées dans le repository lui même:

- JWT secret
- User & Password pour la base de données.

Ces informations ne devraient *jamais* être commitée. Pourquoi ? Parce que cela veut dire que chaque développeur a accès au password de la base de données - et donc la capcité de la supprimer, ou de faire un TRUNCATE, etc.

Nous devons donc trouver un moyen de retirer ces informations du code (.java ou application.properties) pour les mettre... "ailleurs".

Ce "ailleurs" est dans la plupart des cas un éléments au niveau de l'OS - les variables d'environnement.

### Variable d'environnement

Cette notion existe dans chaque OS. En Windows, vous pouvez y accéder via le menu démarrer ("Edit system environment variables"). Ouvrir l'option est intéressant pour voir ce qui s'y trouve déjà. C'est variable selon le système, mais probablement un PATH et un ensemble de variable pour indiquer à windows où trouver certains folder (JAVA_HOME par exemple).

Il est possible de définir ou de lire ces variables depuis le terminal.

En power shell:

```powershell
echo $Env:PATH
$Env:TEXT="Mon texte"
echo $Env:TEXT
```

ou en linux

```bash
echo %PATH%
export TEXT="Mon texte"
echo %TEXT%
```

On peut donc créer une variable d'environnement avec notre password:

```powershell
$Env:DB_PASSWORD=cae
```

et le remplacer dans application.properties:

```properties
spring.application.name=cae_exercices_fiche3

spring.datasource.url=jdbc:postgresql://localhost:5432/cae_db
spring.datasource.username=cae_user
spring.datasource.password=${DB_PASSWORD}
spring.jpa.generate-ddl=true
```

relancez l'application qui devrait fonctionner... mais probablement pas.

Pourquoi ? Les variables d'environnments sont liés à un shell précis - et IntelliJ ne lance pas l'application dans le même.

Pour s'en assurer néanmoins, nous pouvons ouvrir le terminal intégré à IntelliJ et:

```powershell
$Env:DB_PASSWORD=cae
.\mvnw spring-boot:run  
```

La première ligne initialize la variable d'environnement, la seconde lance notre application avec la commande maven.

Les choses devraient maintenant fonctionner - mais ce n'est pas forcément pratique.

Pour configurer ces options directement, vous pouvez éditer votre configuration de lancement "Edit configurations" et ajouter une "configuration property" - ceci va assurer que les variables soient bien configurées avant le lancement.

![configuration properties](/images/configuration.png)

Notre application ne partage donc plus ses secrets - mais leur gestion n'est pas vraiment pratique, d'autant que dans un cas réel nous pourrions avoir une douzaine de variables à configurer:

```bash
# Info DB
DB_NAME=postgres
DB_USER=postgres
DB_HOST=db.mars-central-1.rds.amazonaws.com
DB_PASSWORD=dohiputmypasswordinthecourse

# Email server
INVITE_MAIL_SENDER=support@null.org
FRONTEND_BASE_URL=https://localhost:5173

# Token for an external API (ex: to generate PDF)
PDF_MONKEY_TOKEN=generatingpdfishellbutfrozen

# Token for another external API (ex: openai)
OPENAI_API_KEY=thisisnotarealkey

# Token for file storage (ex: AWS)
AWS_ACCESS_KEY_ID=awsisanofferyoucantunderstand
AWS_SECRET_ACCESS_KEY=butwhenitworkitworkswell

AWS_STORAGE_BUCKET_NAME=bucketbrigade
AWS_S3_REGION_NAME=mars-central-1
...
```

### .env et spring-dotenv

La manière "classique" de gérer ces variables est de créer un petit fichier .env où on va les écrire. L'éléments très important est que ce fichier .env **ne peut en aucun cas être committé**.

Le fichier doit être créer dans src/main/resources (à côté du fichier properties)

Nous allons donc directement l'ajouter au .gitignore:

```bash
...
# .gitignore
.env
```

Il est très important de faire ceci directement. Retirer des informations de l'historique de git est réellement complexe.

On peut alors ajouter nos secret à ce fichier (qui ne quittera donc pas notre machine)

```bash
DB_PASSWORD="cae"
```

Reste à dire à Spring d'aller lire les informations à cet endroit - on va utiliser spring-dotenv (que vous avez installé précédemment) pour cela.

Il suffit de rajouter une paire de lignes dans l'application elle même:

```java
public class CaeExercicesFiche3Application {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext();

        // Add DotenvPropertySource to environment before registering components
        DotenvPropertySource.addToEnvironment(applicationContext.getEnvironment());
        SpringApplication.run(CaeExercicesFiche3Application.class, args);
    }
```

Ceci dit à spring-dotenv de charger la configuration (sans paramètres, il va aller cherdcher un .env exactement là où nous l'avons placé) et de stockers ces valeurs qui sont alors disponible pour l'application.

# Documentation

Nous avons codé un backend et configuré de multiples endpoint - il est maintenant possible de développer un front end pour y ajouter une interface (par exemple avec React).

Le problème côté front end va être de savoir:

- Quels sont les endpoints disponibles ?
- Quel(s) paramètres acceptent il ?

En un mot - de la documentation.

Ceci pourrait être fait manuellement (par exemple dans un fichier markdown) mais cette approche a de nombreux problèmes:

- Manque de standardisation
- Difficile de maintenir la documentation à jour avec le code

Pour résoudre ces problèmes l'industrie a développé des *standards* (comme OpenAPI) et des *outils* (comme Swagger).

## Documentation OpenAPI

[OpenAPI](https://www.openapis.org/what-is-openapi) est une *spécification* indépendante de tout language de programmation ou outils pour définir une API.

[Swagger](https://swagger.io/tools/open-source/) est un *outil* pour nous aider à produire de la documentation sur nos APIs.

Le format sous jacent est souvent soit du JSON soit du YAML - les spécifications évoluant, il y a également des versions (v3 actuellement).

## Générer de la documentation

Nous pourrions créer un document yaml à la main - mais l'écosystème Spring propose quelque chose pour gagner du temps - un package qui va générer cette documentation basé sur nos controllers.

Pour ceci, ajouter une dépendance maven:

```xml
<dependency>
  <groupId>org.springdoc</groupId>
  <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
  <version>2.3.0</version>
</dependency>
```

Rédémarrez votre serveur et allez sur l'url suivante: http://localhost:8080/swagger-ui/index.html

Vous devriez voir la liste complète des endpoints... et plus:

- La liste est groupée par controller
- Chaque endpoint indique son url, mais aussi la méthode HTTP
- Il est possible d'appeller directement un endpoint - Swagger nous indique les paramètres à fournir

Pour aller à la source, allez sur: http://localhost:8080/v3/api-docs (json) ou http://localhost:8080/v3/api-docs.yaml (yaml) pour sortir les documents OpenAPI.

Comment ceci peut il fonctionner ? Notre code contient en fait toutes les informations nécessaires:

Les classes controller ont une annotation @RestController et @RequestMapping qui indiquent les urls:

```java
@RestController
@RequestMapping("/pizzas")
public class PizzaController {
  ...
```

Les endpoints eux mêmes sont également annotés:

```java
    @GetMapping("/{id}")
    public Pizza getPizza(@PathVariable long id) {
```

Le package de Spring Doc récupère ces informations et les converti dans le format OpenAPI - Swagger peut alors générer le UI que nous venons de voir.

### Ajouter de l'information

Bien que l'UI générée soit déjà bien complète, il est possible (et souhaitable) d'ajouter des descriptions aux différents endpoint - ceci se fait via - surprise - des annotations:

```java
    @Operation(summary = "Get a pzza by its id")
    @GetMapping("/{id}")
    public Pizza getPizza(@PathVariable long id) {
```

Redémarrez votre serveur et retournez sur Swagger - la description est maintenant visible dans la documentation.


## Écrire la documentation

Il est parfois utile d'écrire la documentation manuellement au lieu de la générer, par exemple lorsque l'application n'a pas encore été développée ou pour exprimer des besoins.

Swagger propose un éditeur en ligne pour écrire et valider la syntaxe de sa documentation, à l'adresse [editor.swagger.io](https://editor.swagger.io). Il est également possible d'écrire un fichier de documentation OpenAPI directement dans IntelliJ, en sélectionnant "New" > "OpenAPI Specification".

Analysez ensuite le contenu du fichier `documentation.yaml`. Essayez de comprendre la syntaxe d'OpenAPI. Les spécifications complètes de cette syntaxe sont disponibles [sur le site de swagger](https://swagger.io/specification/). Ajoutez les spécifications nécessaires pour la route `login` que vous avez créé, ainsi que les authentifications et autorisations nécessaires.

# Exercices récapitulatifs

## Documentation -> Code

Sur base de la documentation fournie ci-dessous, vous devez développer une API en spring répondant aux besoins demandés.

```yaml
openapi: 3.0.0
info:
  title: API de Gestion de Bibliothèque
  description: API simplifiée pour gérer les livres et les utilisateurs dans une bibliothèque.
  version: 1.0.0

servers:
  - url: http://localhost:8080
    description: Serveur de développement local

paths:
  /books:
    get:
      summary: Récupérer la liste des livres
      parameters:
        - in: query
          name: titleContains
          required: false
          schema:
            type: string
          description: Filtrer les livres dont le titre contient cette chaîne
      responses:
        '200':
          description: Liste des livres
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/Book'

    post:
      summary: Ajouter un nouveau livre
      security:
        - jwt: ['user']
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/Book'
      responses:
        '201':
          description: Livre créé
        '400':
          description: Requête invalide
        '401':
          description: Non autorisé

  /books/{id}:
    get:
      summary: Récupérer un livre par ID
      parameters:
        - in: path
          name: id
          required: true
          schema:
            type: integer
          description: ID du livre
      responses:
        '200':
          description: Livre trouvé
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Book'
        '404':
          description: Livre non trouvé

    delete:
      summary: Supprimer un livre
      security:
        - jwt: ['admin']
      parameters:
        - in: path
          name: id
          required: true
          schema:
            type: integer
          description: ID du livre
      responses:
        '204':
          description: Livre supprimé
        '401':
          description: Non autorisé
        '403':
          description: Accès interdit
        '404':
          description: Livre non trouvé

  /users:
    get:
      summary: Récupérer la liste des utilisateurs
      security:
        - jwt: ['admin']
      responses:
        '200':
          description: Liste des utilisateurs
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/User'
        '401':
          description: Non autorisé
        '403':
          description: Accès interdit

    post:
      summary: Ajouter un nouvel utilisateur
      security:
        - jwt: ['admin']
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/User'
      responses:
        '201':
          description: Utilisateur créé
        '400':
          description: Requête invalide
        '401':
          description: Non autorisé

components:
  schemas:
    Book:
      type: object
      properties:
        id:
          type: integer
        title:
          type: string
        author:
          type: string
        publishedYear:
          type: integer

    User:
      type: object
      properties:
        id:
          type: integer
        username:
          type: string
        email:
          type: string
        role:
          type: string
          enum: ['admin', 'user']

  securitySchemes:
    jwt:
      type: http
      scheme: bearer
      bearerFormat: JWT
      description: Authentification par token JWT
```

Note: Cette documentation a été générée avec une IA. Que pensez vous de la qualité de cette documentation, notamment dans les descriptions fournies ? 

Une documentation ne peut être utile que si elle est parfaitement correcte et répond à un maximum des questions que peut se poser un utilisateur.

## Code -> Documentation

Créez la documentation de l'API disponible à l'adresse [github.com/e-vinci/cae-exercices-exemple](https://github.com/e-vinci/cae-exercices-exemple). N'hésitez pas à récupérer cette API sur votre machine et à la tester pour vous assurer de bien comprendre son fonctionnement.
