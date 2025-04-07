<p align="center">
    <img src="img/rsa-kryptosysteme.png" alt="RSA Portada" width="400"  />
</p>

# ¿Qué es RSA?
RSA es un algoritmo de cifrado asímétrico y, por tanto, trabaja con dos claves. Una clave pública y una clave privada.

Un texto plano, antes de transmitirlo al destinatario, podríamos cifrarlo con la clave pública. El receptor del mensaje, usando la clave privada, podría descifrar el mensaje. Esto también funciona a la inversa, es decir, si ciframos el mensaje con la clave privada podrá ser descifrado mediante la clave pública.

<p align="center">
    <img src="img/criptografiaAsimetrica.png" alt="Transmision mensaje RSA" width="400"  />
</p>

Sin embargo, si alguien mal intencionado interceptara la comunicación, no podría ver el mensaje dado que no poseerá la clave para descifrarlo, salvo que consiga romper el cifrado. Esto me lleva al siguiente punto, la importancia de la longitud de las claves RSA.

# POC: Importantacia de la longitud de las claves RSA



