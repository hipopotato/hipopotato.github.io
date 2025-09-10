# Recursos do Projeto

## Vídeo – Demonstração com Manim

<iframe width="560" height="315"
    src="https://www.youtube.com/embed/zSOAJamGVpg"
    title="Vídeo Manim de ladrilhamento"
    frameborder="0"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
    allowfullscreen>
</iframe>

---

## Código Manim – Ladrilhamento 12-4-6

```python
from manim import *
import numpy as np

class Tiling124(Scene):
    def construct(self):
        # --- Configurações ---
        self.camera.background_color = WHITE  # 1. Fundo branco
        L = 1.0  # comprimento do lado dos polígonos

        all_polygons = VGroup()  # grupo para armazenar todos os polígonos

        # --- Função auxiliar para criar e posicionar polígonos ---
        def attach_polygon(n_sides, color, target_side, reference_center):
            radius = L / (2 * np.sin(PI / n_sides))
            poly = RegularPolygon(n=n_sides, radius=radius)
            poly.set_fill(color, opacity=1)
            poly.set_stroke(BLACK, width=2)  # 1. Borda preta

            # alinhamento do lado
            v = poly.get_vertices()
            p1, p2 = v[0], v[1]

            # Translação
            shift_vec = target_side[0] - p1
            poly.shift(shift_vec)
            p2_new = p2 + shift_vec

            # Rotação
            angle_target = np.angle(complex(*(target_side[1] - target_side[0])[:2]))
            angle_current = np.angle(complex(*(p2_new - target_side[0])[:2]))
            poly.rotate(angle_target - angle_current, about_point=target_side[0])

            # Garantir que fique para fora
            mid_side = (target_side[0] + target_side[1]) / 2
            center_poly = poly.get_center()
            outward_vec_ref = mid_side - reference_center
            outward_vec_poly = center_poly - mid_side
            if np.dot(outward_vec_poly, outward_vec_ref) < 0:
                poly.rotate(PI, about_point=mid_side)

            return poly

        # --- Função para evitar duplicatas ---
        def is_duplicate(poly, group):
            if not group:
                return False
            for p in group:
                if np.linalg.norm(poly.get_center() - p.get_center()) < 0.01:
                    return True
            return False

        # --- Dodecágono central ---
        dodecagon = RegularPolygon(n=12, radius=L / (2*np.sin(PI/12)))
        dodecagon.set_fill(BLUE, opacity=1)
        dodecagon.set_stroke(BLACK, width=2)  # 1. Borda preta
        all_polygons.add(dodecagon)
        self.play(Create(dodecagon), run_time=1.5)

        # --- Primeira camada: quadrados e hexágonos nos vértices ---
        first_layer = VGroup()
        vertices = dodecagon.get_vertices()
        for i in range(12):
            v_prev = vertices[i-1]
            v_curr = vertices[i]
            v_next = vertices[(i+1)%12]

            if i % 2 == 0:
                square = attach_polygon(4, RED, (v_curr, v_next), dodecagon.get_center())
                first_layer.add(square)
                hexagon = attach_polygon(6, GREEN, (v_prev, v_curr), dodecagon.get_center())
                first_layer.add(hexagon)
            else:
                hexagon = attach_polygon(6, GREEN, (v_curr, v_next), dodecagon.get_center())
                first_layer.add(hexagon)
                square = attach_polygon(4, RED, (v_prev, v_curr), dodecagon.get_center())
                first_layer.add(square)

        all_polygons.add(*first_layer)
        self.play(Create(first_layer), run_time=2)

        # --- Segunda camada: expansão ---
        second_layer = VGroup()
        second_layer_polys_list = []
        for poly in first_layer:
            vertices = poly.get_vertices()
            num_sides = len(vertices)
            center_ref = dodecagon.get_center()

            min_dist = float('inf')
            attached_edge_index = 0
            for i in range(num_sides):
                mid = (vertices[i] + vertices[(i+1)%num_sides])/2
                dist = np.linalg.norm(mid - center_ref)
                if dist < min_dist:
                    min_dist = dist
                    attached_edge_index = i

            if num_sides == 4:
                free_edge_index = (attached_edge_index + 2) % 4
                side_free = (vertices[free_edge_index], vertices[(free_edge_index+1)%4])
                new_dode = attach_polygon(12, BLUE, side_free, poly.get_center())
                if not is_duplicate(new_dode, all_polygons):
                    second_layer.add(new_dode)
                    all_polygons.add(new_dode)
                    second_layer_polys_list.append(new_dode)

                for offset in [1, 3]:
                    lateral_edge_index = (attached_edge_index + offset) % 4
                    side_lateral = (vertices[lateral_edge_index], vertices[(lateral_edge_index+1)%4])
                    new_hexagon = attach_polygon(6, GREEN, side_lateral, poly.get_center())
                    if not is_duplicate(new_hexagon, all_polygons):
                        second_layer.add(new_hexagon)
                        all_polygons.add(new_hexagon)
                        second_layer_polys_list.append(new_hexagon)

            elif num_sides == 6:
                opposite_edge_index = (attached_edge_index + 3) % 6
                side_opposite = (vertices[opposite_edge_index], vertices[(opposite_edge_index+1)%6])
                new_square = attach_polygon(4, RED, side_opposite, poly.get_center())
                if not is_duplicate(new_square, all_polygons):
                    second_layer.add(new_square)
                    all_polygons.add(new_square)
                    second_layer_polys_list.append(new_square)

        self.play(Create(second_layer), run_time=3)
        self.wait(0.5)

        # --- 3ª camada: completando os dodecágonos externos ---
        third_layer = VGroup()
        for base_poly in second_layer_polys_list:
            if len(base_poly.get_vertices()) == 12:
                vertices = base_poly.get_vertices()
                min_dist = float('inf')
                attached_edge_index = 0
                for i in range(12):
                    mid = (vertices[i] + vertices[(i+1)%12])/2
                    dist = np.linalg.norm(mid)
                    if dist < min_dist:
                        min_dist = dist
                        attached_edge_index = i
                for offset in range(1, 12):
                    edge_index = (attached_edge_index + offset) % 12
                    side = (vertices[edge_index], vertices[(edge_index + 1) % 12])
                    if offset % 2 != 0:  # Ímpar: hexágono
                        new_poly = attach_polygon(6, GREEN, side, base_poly.get_center())
                    else:  # Par: quadrado
                        new_poly = attach_polygon(4, RED, side, base_poly.get_center())
                    if not is_duplicate(new_poly, all_polygons):
                        third_layer.add(new_poly)
                        all_polygons.add(new_poly)

        self.play(Create(third_layer), run_time=4)
        self.wait(3)
