Plants vs Zombies 3D – README

Este proyecto es una recreación en 3D del juego Plants vs Zombies, desarrollada con Python, Pygame y OpenGL. Presenta un entorno totalmente tridimensional, con plantas, zombis, soles y animaciones básicas.

Requisitos

Python 3.8 o superior

Librerías:

pygame

PyOpenGL

PyOpenGL_accelerate (opcional, pero recomendado)

Instalación:

pip install pygame PyOpenGL PyOpenGL_accelerate

Cómo ejecutar

Ejecuta el archivo principal:

python nombre_del_archivo.py

Controles

1: Seleccionar Girasol

2: Seleccionar Lanzaguisantes

3: Seleccionar Nuez

Click izquierdo: Colocar planta o recolectar sol

Escape: Cancelar selección o salir

R: Reiniciar cuando el juego termine

Mecánicas principales

Girasol: genera soles con el tiempo.

Lanzaguisantes: ataca zombis en su misma fila.

Nuez: actúa como defensa con mucha vida.

Zombis: avanzan por las filas, comen plantas y generan dificultad progresiva.

Soles: aparecen del cielo o los producen los girasoles.

Características

Renderizado en 3D con OpenGL.

Sombreado, animaciones y efectos simples.

Colisiones entre guisantes y zombis.

Sistema de oleadas y aumento de dificultad.

Interfaz 2D sobrepuesta (HUD).

Objetivo

Evitar que los zombis crucen el tablero. Si uno llega al borde izquierdo, el juego termina.
