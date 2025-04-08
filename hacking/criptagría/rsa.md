<p align="center">
    <img src="img/rsa-kryptosysteme.png" alt="RSA Portada" width="400"  />
</p>

# ¿Qué es RSA?
RSA es un algoritmo de cifrado asímétrico y, por tanto, trabaja con dos claves. Una clave pública y una clave privada.

Un mensaje en texto plano, antes de transmitirlo al destinatario, podríamos cifrarlo con la clave pública. El receptor del mensaje, usando la clave privada, podría descifrar el mensaje. Esto también funciona a la inversa, es decir, si ciframos el mensaje con la clave privada podrá ser descifrado mediante la clave pública.

Espero que con la siguiente imagen, quede un poco más claro este concepto:
<p align="center">
    <img src="img/criptografiaAsimetrica.png" alt="Transmision mensaje RSA" width="300"  />
</p>

Sin embargo, si alguien mal intencionado interceptara la comunicación, no podría ver el mensaje dado que no poseerá la clave para descifrarlo, salvo que consiga romper el cifrado. Haremos una prueba de concepto, más adelante, donde veremos la importancia de tener claves con una longitud de adecuada para que no sea viable (o al menos dificultar al máximo) la ruptura de ese cifrado.
<br>

# ¿Cómo podemos generar nuestra clave privada?
A continuación, generaremos nuestra clave privada de forma "manual" (sin usar openssl, por ejemplo). Esto permitirá conocer ciertos conceptos. Para ello, necesitaremos los valores de una serie de variables:

- "p" y "q": Son dos números primos.
- "n": Que es el resultado de multiplicar "p" y "q".
- "e": Suele ser un valor fijo, 65537
- "m": Que se obtiene de aplicar la fórmula n-(p+q-1)
- "d": Que es el resultado de realizar la operación modular multiplicativa inversa de "e" y "m". A continuación, veremos cómo podemos generar nuestra clave privada mediante Python y la librería Crypto.

Este sería el código completo de nuestro script, que genera nuestra clave privada.
```bash
#/usr/bin/env python3

from Crypto.PublicKey import RSA
from pwn import *

# Numeros primos
p=2425967623052370772757633156976982469681
q=1451730470513778492236629598992166035067

# n se obtiene del producto de "p" y "q"
n=p*q

# e es una valor fijo
e=65537

# m la obtenemos de la siguiente formula
m=n-(p+q-1)

# Definimos nuestra función modular multiplicativa inversa
# https://stackoverflow.com/questions/4798654/modular-multiplicative-inverse-function-in-python
def egcd(a, b):
    if a == 0:
        return (b, 0, 1)
    else:
        g, y, x = egcd(b % a, a)
        return (g, x - (b // a) * y, y)

def modinv(a, m):
    g, x, y = egcd(a, m)
    if g != 1:
        raise Exception('modular inverse does not exist')
    else:
        return x % m

# Función modular multiplicativa inversa
d=modinv(e, m)

key=RSA.construct((n, e, d, p, q))
print(key.exportKey().decode())
```

# ¿Cómo podemos generar nuestra clave pública a raiz de nuestra clave privada?
La clave privada generada anteriormente por nuestro script, la guardaremos en un fichero llamado private.pem.
```bash
❯ /bin/cat private.pem                                                                                                                                                  
-----BEGIN RSA PRIVATE KEY-----
MIGsAgEAAiEealEX7v6ob1L6oQgdQ7N5oMTL5L3zyY1ednofzGYveYsCAwEAAQIh
DJwDFqsMhIPyoNlV5dwFciiNPXIxw4Tzip/5dnJ7QPnBAhEHIRhecocAmQYt95EV
Y02sMQIRBEQpHlGz6l/RZnPpVnSwHnsCEQKaxbNjtgyy2z4Z7yhYMj1RAhAxbYL3
w6K5z2pJmKfXBBl9AhEB+gEsX+Hc0oTV+aL00Zl5iQ==
-----END RSA PRIVATE KEY-----

```

Con el siguiente comando de Linux, a partir de nuestra clave privada, podemos crear la clave pública y la guardaremos un fichero llamado public.pem
```bash
❯ openssl rsa -in private.pem -pubout > public.pem                                                                                                                      
writing RSA key
❯ /bin/cat public.pem                                                                                                                                                   
-----BEGIN PUBLIC KEY-----
MDwwDQYJKoZIhvcNAQEBBQADKwAwKAIhHmpRF+7+qG9S+qEIHUOzeaDEy+S988mN
XnZ6H8xmL3mLAgMBAAE=
-----END PUBLIC KEY-----
```

# ¿Cómo podemos cifrar un mensaje con nuestra clave pública?
Vamos a crear un fichero de texto, con el contenido "Mensaje secreto".
```bash
echo "Mensaje secreto" > mensaje.txt  
```

Para cifrarlo, ejecuraremos el siguiente comando:
```bash
openssl pkeyutl --encrypt --inkey public.pem --pubin --in mensaje.txt --out mensaje_secreto.txt
```

El comando anterior, nos habrá generado un nuevo fichero llamado mensaje_secreto.txt. Si intentamos ver su contenido, veremos que no somos capaces de leerlo.
```bash
❯ /bin/cat mensaje_secreto.txt                                                                                                                                          
X�d検�d!�(#     
```

# POC: ¿Cómo podemos descifrar nuestro mensaje si no tenemos la clave pública?
Partimos de un escenario, en el que solo poseemos la clave pública (public.pem) y el mensaje cifrado (mensaje_secreto.txt). 
Para poder descifrar el mensaje, debemos conseguir la clave privada. </br>

Los valores de "n" y "e", necesarios para poder construir la clave privada, los podemos averiguar de la clave pública.
```bash
❯ python3                                                                                                                                                               
Type "help", "copyright", "credits" or "license" for more information.
>>> from Crypto.PublicKey import RSA
>>> f=open("public.pem")
>>> key=RSA.importKey(f.read())
>>> print(key.n)
3521851118865011044136429217528930691441965435121409905222808922963363310303627
>>> print(key.e)
65537
>>> 
```

Teniendo esto en mente, podemos apoyarnos en la libreria SageMath para obtener los valores de "p" y "q".
> [!TIP]
> SageMath, conocido anteriormente como Sage, es un sistema algebraico computacional que destaca por estar construido sobre paquetes matemáticos ya contrastados como NumPy, Sympy, PARI/GP o Maxima y por acceder a sus potencias combinadas a través de un lenguaje común basado en Python.
> https://sagemanifolds.obspm.fr/index.html

Este sería el script resultante.

```bash
#/usr/bin/env python3
from Crypto.PublicKey import RSA
from sage.all import *
import time

# inicio
start_time = time.time()

# Leemos la clave publica
f=open("/home/kali/HTB/public.pem")
key=RSA.importKey(f.read())

# Obtenemos los valores de n y e
n=key.n
e=key.e

# Con la funcion factor, obtenemos los valores de "p" y "q"
p, q =  factor(n)

# Los convertimos en en enteros para poder operar con ellos.
p=int(p[0])
q=int(q[0])

print(p)
print(q)

# m la obtenemos de la siguiente formula
m=n-(p+q-1)

def egcd(a, b):
    if a == 0:
        return (b, 0, 1)
    else:
        g, y, x = egcd(b % a, a)
        return (g, x - (b // a) * y, y)

def modinv(a, m):
    g, x, y = egcd(a, m)
    if g != 1:
        raise Exception('modular inverse does not exist')
    else:
        return x % m


# Función modular multiplicativa inversa
d=modinv(e, m)

key=RSA.construct((n, e, d, p, q))
print(key.exportKey().decode())

# fin y calculamos tiempo que nos ha llevado
end_time = time.time()
print(f"[+] Tiempo empleado: {end_time - start_time} segundos·")
```
</br> 
Su ejecucción, nos devolvería la clave privada en menos de 5 minutos.

```bash
❯python3 decrypt_message.py 
1451730470513778492236629598992166035067
2425967623052370772757633156976982469681
-----BEGIN RSA PRIVATE KEY-----
MIGsAgEAAiEealEX7v6ob1L6oQgdQ7N5oMTL5L3zyY1ednofzGYveYsCAwEAAQIh
DJwDFqsMhIPyoNlV5dwFciiNPXIxw4Tzip/5dnJ7QPnBAhEERCkeUbPqX9Fmc+lW
dLAeewIRByEYXnKHAJkGLfeRFWNNrDECEDFtgvfDornPakmYp9cEGX0CEQKaxbNj
tgyy2z4Z7yhYMj1RAhEDFVxlFotvYZuSQ50G2V9JCQ==
-----END RSA PRIVATE KEY-----
[+] Tiempo empleado: 264.9520308971405 segundos·
```

Guardamos la clave privada que el script nos ha generado, en un fichero llamado private.pem. Por último, solo tenemos que usar la clave privada para descifrar el mensaje y ver su contenido.
```bash
❯ openssl pkeyutl -decrypt -inkey private.pem -in mensaje_secreto.txt                                                                                                   
Mensaje secreto
```











