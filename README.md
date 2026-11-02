import pygame
import random
import math
import sys

# --- Constantes de la Interfaz ---
ANCHO, ALTO = 600, 800
ANCHO_BLOQUE = 80
ALTO_BLOQUE = 30
INICIO_X = (ANCHO - (ANCHO_BLOQUE * 3)) // 2
BASE_Y = ALTO - 50
GRAVEDAD = 0.5  # Fuerza con la que caen los bloques

# Colores (RGB)
COLOR_FONDO = (135, 206, 235)  
COLOR_BLOQUE = (210, 180, 140) 
COLOR_BORDE = (139, 69, 19)    
COLOR_SUELO = (34, 139, 34)    
COLOR_TEXTO = (0, 0, 0)

# --- Estructura de Datos: Lista Doblemente Enlazada ---
class NodoNivel:
    def __init__(self):
        self.bloques = [True, True, True] 
        self.siguiente = None 
        self.anterior = None  

class TorreJenga:
    def __init__(self, niveles_iniciales=10):
        self.base = None
        self.cima = None
        self.niveles = 0
        
        for _ in range(niveles_iniciales):
            self.agregar_piso_completo()

    def agregar_piso_completo(self):
        nuevo_nivel = NodoNivel()
        if not self.base:
            self.base = nuevo_nivel
            self.cima = nuevo_nivel
        else:
            self.cima.siguiente = nuevo_nivel
            nuevo_nivel.anterior = self.cima
            self.cima = nuevo_nivel
        self.niveles += 1

    def obtener_nivel(self, indice):
        actual = self.base
        contador = 0
        while actual:
            if contador == indice:
                return actual
            contador += 1
            actual = actual.siguiente
        return None

    def agregar_bloque_arriba(self):
        if not self.cima.bloques[0]:
            self.cima.bloques[0] = True
        elif not self.cima.bloques[1]:
            self.cima.bloques[1] = True
        elif not self.cima.bloques[2]:
            self.cima.bloques[2] = True
        else:
            nuevo_nivel = NodoNivel()
            nuevo_nivel.bloques = [True, False, False]
            self.cima.siguiente = nuevo_nivel
            nuevo_nivel.anterior = self.cima
            self.cima = nuevo_nivel
            self.niveles += 1

    def verificar_estabilidad(self):
        actual = self.base
        while actual and actual != self.cima:
            b = actual.bloques
            # Condiciones de derrumbe
            if b == [False, False, False] or b == [True, False, False] or b == [False, False, True]:
                return False 
            actual = actual.siguiente
        return True

# --- Funciones Gráficas ---
def dibujar_entorno(pantalla):
    pantalla.fill(COLOR_FONDO)
    pygame.draw.rect(pantalla, COLOR_SUELO, (0, BASE_Y, ANCHO, ALTO - BASE_Y))

def dibujar_torre_estatica(pantalla, torre):
    actual = torre.base
    nivel_idx = 0
    
    while actual:
        y_pos = BASE_Y - (nivel_idx + 1) * ALTO_BLOQUE
        for i in range(3):
            if actual.bloques[i]:
                x_pos = INICIO_X + i * ANCHO_BLOQUE
                rect_bloque = pygame.Rect(x_pos, y_pos, ANCHO_BLOQUE, ALTO_BLOQUE)
                pygame.draw.rect(pantalla, COLOR_BLOQUE, rect_bloque)
                pygame.draw.rect(pantalla, COLOR_BORDE, rect_bloque, 2)
        actual = actual.siguiente
        nivel_idx += 1

def extraer_bloques_para_animacion(torre):
    """ Desarma la lista enlazada y convierte los nodos en físicas independientes """
    bloques_fisicos = []
    actual = torre.base
    nivel_idx = 0
    
    while actual:
        y_pos = BASE_Y - (nivel_idx + 1) * ALTO_BLOQUE
        for i in range(3):
            if actual.bloques[i]:
                x_pos = INICIO_X + i * ANCHO_BLOQUE
                # Le damos velocidades y rotaciones caóticas para simular la caída
                bloques_fisicos.append({
                    'x': x_pos,
                    'y': y_pos,
                    'vx': random.uniform(-4, 4),      # Salen volando hacia los lados
                    'vy': random.uniform(-2, 2),      # Pequeño rebote inicial
                    'rotacion': 0,
                    'vel_rotacion': random.uniform(-10, 10) # Giran en el aire
                })
        actual = actual.siguiente
        nivel_idx += 1
    return bloques_fisicos

def dibujar_y_actualizar_caida(pantalla, bloques_fisicos):
    for b in bloques_fisicos:
        # Físicas
        b['vy'] += GRAVEDAD  # La gravedad los empuja hacia abajo
        b['x'] += b['vx']
        b['y'] += b['vy']
        b['rotacion'] += b['vel_rotacion']
        
        # Crear superficie para poder rotar el rectángulo
        surf = pygame.Surface((ANCHO_BLOQUE, ALTO_BLOQUE), pygame.SRCALPHA)
        surf.fill(COLOR_BLOQUE)
        pygame.draw.rect(surf, COLOR_BORDE, surf.get_rect(), 2)
        
        # Rotar y centrar
        surf_rotada = pygame.transform.rotate(surf, b['rotacion'])
        rect_rotado = surf_rotada.get_rect(center=(b['x'] + ANCHO_BLOQUE/2, b['y'] + ALTO_BLOQUE/2))
        
        # Dibujar en pantalla
        pantalla.blit(surf_rotada, rect_rotado)

# --- Bucle Principal ---
def iniciar_jenga():
    pygame.init()
    pantalla = pygame.display.set_mode((ANCHO, ALTO))
    pygame.display.set_caption("Jenga 2D - Efecto de Caída (Lista Doblemente Enlazada)")
    fuente = pygame.font.SysFont("arial", 24, bold=True)
    reloj = pygame.time.Clock()

    torre = TorreJenga(12)
    jugador_actual = 1
    estado_juego = "JUGANDO"
    bloques_cayendo = []

    while True:
        for evento in pygame.event.get():
            if evento.type == pygame.QUIT:
                pygame.quit()
                sys.exit()

            if evento.type == pygame.MOUSEBUTTONDOWN and estado_juego == "JUGANDO":
                pos_x, pos_y = evento.pos
                
                if INICIO_X <= pos_x <= INICIO_X + (3 * ANCHO_BLOQUE) and pos_y <= BASE_Y:
                    columna = int((pos_x - INICIO_X) // ANCHO_BLOQUE)
                    nivel_clickeado = int((BASE_Y - pos_y) // ALTO_BLOQUE)
                    
                    if nivel_clickeado < torre.niveles - 1: 
                        nodo_objetivo = torre.obtener_nivel(nivel_clickeado)
                        
                        if nodo_objetivo and nodo_objetivo.bloques[columna]:
                            nodo_objetivo.bloques[columna] = False
                            torre.agregar_bloque_arriba()
                            
                            if not torre.verificar_estabilidad():
                                estado_juego = "DERRUMBE"
                                # Convertimos la lista enlazada en objetos físicos
                                bloques_cayendo = extraer_bloques_para_animacion(torre)
                            else:
                                jugador_actual = 2 if jugador_actual == 1 else 1
                                
            if evento.type == pygame.KEYDOWN:
                if evento.key == pygame.K_r and estado_juego == "DERRUMBE":
                    torre = TorreJenga(12)
                    jugador_actual = 1
                    estado_juego = "JUGANDO"
                    bloques_cayendo = []

        dibujar_entorno(pantalla)

        if estado_juego == "JUGANDO":
            dibujar_torre_estatica(pantalla, torre)
            texto = fuente.render(f"Turno: Jugador {jugador_actual}", True, COLOR_TEXTO)
            pantalla.blit(texto, (20, 20))
        elif estado_juego == "DERRUMBE":
            dibujar_y_actualizar_caida(pantalla, bloques_cayendo)
            texto = fuente.render(f"¡TORRE DERRUMBADA! Gana Jug. {3 - jugador_actual}", True, (255, 0, 0))
            pantalla.blit(texto, (20, 20))
            texto_re = fuente.render("Presiona 'R' para reiniciar", True, COLOR_TEXTO)
            pantalla.blit(texto_re, (20, 60))

        pygame.display.update()
        reloj.tick(60)

if __name__ == "__main__":
    iniciar_jenga()
