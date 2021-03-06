---
tags:
- Desarrollo web
- Programación web
- Framework
- Linux
categories:
- Symfony
title: Login de usuario en Symfony 3.
---

En este post se mostrará un ejemplo sobre como crear un [login](http://symfony.com/doc/current/security/form_login_setup.html) en Symfony 3.

## Rutas

En el *bundle* que queramos añadir el login se modifica el archivo *Resources/config/routing.yml* para añadir las rutas necesarias para la creación del login:

```yml
login:
    path: /login
    defaults: { _controller: FilmBundle:User:login }

login_check:
    path: /login_check

logout:
    path: /logout
```

## Security

Es necesario modificar el archivo *app/config/security.yml* de la siguiente manera:

```yml
security:
    encoders:
        FilmBundle\Entity\User:
            algorithm: bcrypt #Algoritmo de cifrado para la contraseña
            cost: 4
            
    providers:
        our_db_provider: 
            entity:
                class: FilmBundle:User #Clase con la que se realiza el login (Usuario)
                property: email #Campo con el que se identifica (email)

    firewalls:
        dev:
            pattern: ^/(_(profiler|wdt)|css|images|js)/
            security: false

        main:
            anonymous: ~
            provider: our_db_provider 
            form_login:
                login_path: /login
                check_path: /login_check
            logout:
                path: /logout
                target: /login #Ruta a la que se dirige al cerrar sesión (vuelve al login)
```

## User.php

El login se realizará con la clase *Entity/User.php* por lo que para que funcione la clase debe de implementar *UserInterface*: 

```javascript
use Symfony\Component\Security\Core\User\UserInterface;

class User implements UserInterface
{
...
}
```

Además hay que añadir los siguientes métodos:

```javascript
public function getUserName(){
    return $this->email; //Ya que se logea con el email
}

public function getSalt(){
    return null;
}

public function getRoles(){
    return array($this->getRole());
}

public function eraseCredentials(){}
```

## Controlador

En el controlador, como viene en la [documentación](http://symfony.com/doc/current/security/form_login_setup.html) se llama a la vista pasandole el error en caso de que se produzca y el último usuario introducido.

```javascript
<?php

namespace FilmBundle\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\Controller;
use Symfony\Component\HttpFoundation\Request;

class UserController extends Controller
{
    public function loginAction(Request $request){
        $authenticationUtils = $this->get("security.authentication_utils");
        $error = $authenticationUtils->getLastAuthenticationError();
        $lastUsername = $authenticationUtils->getLastUsername();

        return $this->render("FilmBundle:User:login.html.twig", array(
            "error" => $error,
            "last_username" => $lastUsername
        ));
    }
}
```

## Vista

En la vista se muestra el login:

```html
<!--Este if es solo para comprobar si funciona el Login, si funciona, eliminarlo-->
{% raw %}
{% if app.user != null %}
	{{ dump(app.user) }}
{% endif %}
{% endraw %}
<h2>Login:</h2>
<div class="col-lg-4">
	<form action="{% raw %}{{path('login_check')}}{% endraw %}" method="post">
		<label>Email:</label>
		<input type="email" id="username" name="_username" value="{{ last_username }}" class="form-control" />
		<label>Password:</label>
		<input type="password" id="password" name="_password" class="form-control" />
		<br/>
		<input type="submit" value="Entrar" class="btn btn-success"/>
		<!-- Al envíar el formulario redirije a /login -->
		<input type="hidden" name="_target_path" value="/login" />
	</form>
</div>
```


![CNAME]({{ site.baseurl }}/images/login01.png "LOGIN")

## Cifrar contraseñas

En el registro de usuarios es importante que cuando se almacenen contraseñas de usuarios se hagan cifradas. Para conseguir esto se añade en el controlador:

```javascript
$factory = $this->get("security.encoder_factory"); 
$encoder = $factory->getEncoder($user);
//Se almacena en password la contraseña cifrada
$password = $encoder->encodePassword($form->get("password")->getData(), $user->getSalt());    
```

## Control de acceso

Es posible permitir o denegar el acceso a una ruta según sea el rol del usuario (*ROLE_USER, ROLE_ADMIN...*). Esto se indica en el archivo *app/config/security.yml*. 

```yml
security:
  encoders:
    ...
   
  providers:
    ...
    
  firewalls:
    ...
    
  access_control:
    - { path: ^/film, roles: ROLE_ADMIN }
    - { path: ^/book, roles: ROLE_ADMIN }
```

En este ejemplo, a las rutas */film* y */book* únicamente pueden acceder los usuarios logeados con el rol ADMIN. Es posible hacer que una ruta sea accesible por varios roles:

```yml
access_control:
  - { path: ^/film, roles: [ROLE_USER, ROLE_ADMIN] }
```

Finalmente podemos mostrar o ocultar cierta sección de la página dependiendo del rol del usuario:

```yml
//Si está logeado como usuario
{% raw %}{% if is_granted('ROLE_USER') %}
  ...
{% endif %}  {% endraw %}
```
