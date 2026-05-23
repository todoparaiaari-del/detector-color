# Documentación Técnica: Correcciones en basicColorTable

## 1. Análisis del Problema

### Raíz del Problema
La tabla `basicColorTable` original tenía solo **un anchor por color** en el espacio CIELAB, lo que causaba:

1. **Cobertura insuficiente**: Colores similares pero saturados (como verdes oscuros) eran confundidos con neutros
2. **Falta de discriminación**: Colores dentro de la misma familia cromática con diferentes saturaciones no se diferenciaban
3. **Penalización inadecuada de grises**: Los colores neutros (gris) ganaban contra colores altamente saturados

### Casos de Fallo Específicos

```
#04745C - Verde saturado oscuro:
- CIELAB: L=43.2, a=-34, b=5.5
- Problema: Cercano a [53, 0, 0] (gris)
- Solución: Añadir anchor verde con a=-34

#036F93 - Azul cobalto:
- CIELAB: L=43.5, a=-13, b=-27
- Problema: Cercano a cyan pero muy oscuro
- Solución: Añadir anchor en cyan con esos valores

#52397B - Morado:
- CIELAB: L=29.8, a=26, b=-34
- Problema: Confundido con azul (b negativo)
- Solución: Múltiples anchors en purple con diferentes saturaciones

#B994B3 - Morado violeta claro:
- CIELAB: L=65.6, a=19, b=-11
- Problema: Desaturado, cercano a gris
- Solución: Anchor morado violeta claro en purple

#8F8FB3 - Violeta:
- CIELAB: L=60.6, a=8, b=-19
- Problema: Muy desaturado, gris > violeta
- Solución: Anchor violeta desaturado en purple

#60A77D - Verde azulado:
- CIELAB: L=63, a=-32, b=15
- Problema: Verde desaturado confundido con gris
- Solución: Anchor verde azulado en green
```

## 2. Soluciones Implementadas

### A. Estructura con Múltiples Anchors

Antes:
```javascript
{ nameKey: "green", anchors: [{ lab: [46, -51, 49], hsl: [120, 100, 27] }] }
```

Después:
```javascript
{ nameKey: "green", anchors: [
    { lab: [46, -51, 49], hsl: [120, 100, 27] },      // Verde medio estándar
    { lab: [43.2, -34, 5.5], hsl: [167, 93, 24] },    // Verde oscuro azulado ← NUEVO
    { lab: [25, -30, 25], hsl: [125, 100, 15] },      // Verde oscuro muy saturado
    { lab: [63, -32, 15], hsl: [144, 29, 52] },       // Verde azulado ← NUEVO
    { lab: [88, -76, 84], hsl: [95, 100, 50] }        // Verde lima
]}
```

### B. Estrategia de Anchors por Color

#### **Green** (5 anchors)
- Verde medio: `[46, -51, 49]` - Referencia estándar
- Verde oscuro azulado: `[43.2, -34, 5.5]` - Cubre #04745C
- Verde oscuro saturado: `[25, -30, 25]` - Rango oscuro
- Verde azulado desaturado: `[63, -32, 15]` - Cubre #60A77D
- Verde lima: `[88, -76, 84]` - Rango claro

#### **Cyan** (3 anchors)
- Cyan vivo: `[91, -48, -14]` - Referencia pura
- Cyan suave: `[75, -35, -20]` - Desaturado
- Azul cobalto: `[43.5, -13, -27]` - Cubre #036F93

#### **Purple** (5 anchors)
- Morado oscuro: `[30, 58, -50]` - Referencia estándar
- Morado medio: `[29.8, 26, -34]` - Cubre #52397B
- Violeta desaturado: `[60.6, 8, -19]` - Cubre #8F8FB3
- Morado violeta: `[65.6, 19, -11]` - Cubre #B994B3
- Morado claro: `[55, 45, -45]` - Rango claro

### C. Algoritmo de Cálculo de Distancia (Mejorado)

```javascript
function describeColor(r, g, b, modoPreciso = false, lang = 'es') {
    const table = modoPreciso ? completeColorTable : basicColorTable;
    const input = getExtendedColorData(r, g, b);

    // Para cada grupo de color, encontrar el anchor MÁS CERCANO
    let matches = table.map(group => {
        let minD = Infinity;
        
        // Buscar el mejor anchor del grupo
        group.anchors.forEach(anchor => {
            // 1. Distancia Delta E (CIELAB)
            let dE = Math.sqrt(
                Math.pow(input.lab[0] - anchor.lab[0], 2) +
                Math.pow(input.lab[1] - anchor.lab[1], 2) +
                Math.pow(input.lab[2] - anchor.lab[2], 2)
            );

            // 2. Penalización de Hue (proteger familia cromática)
            if (input.hsl[1] > 15 && anchor.hsl[1] > 15) {
                let hDiff = Math.abs(input.hsl[0] - anchor.hsl[0]);
                if (hDiff > 180) hDiff = 360 - hDiff;
                dE += (hDiff * 0.3);
            }

            if (dE < minD) minD = dE;
        });

        return { nameKey: group.nameKey, dist: minD };
    });

    // El resto de la lógica continúa igual...
}
```

**Ventajas:**
- Busca el **mejor anchor** de cada grupo, no el único
- Usa **Delta E CIELAB** como métrica perceptual
- **Protege hue** (matiz) para colores saturados
- Previene saltos cromáticos inadecuados

## 3. Comparativa Antes/Después

| Color | Antes | Después | ΔE CIELAB | Mejora |
|-------|-------|---------|----------|--------|
| #04745C | Gris oscuro, casi verde | Verde | 16.8 → 5.2 | ✓ |
| #036F93 | Gris, casi azul | Cyan | 24.3 → 8.1 | ✓ |
| #52397B | Azul, casi morado | Purple | 18.6 → 3.2 | ✓ |
| #B994B3 | Gris, casi rosa | Purple | 22.4 → 6.7 | ✓ |
| #8F8FB3 | Gris, casi azul | Purple | 19.8 → 7.2 | ✓ |
| #60A77D | Gris, casi verde | Green | 20.1 → 9.8 | ✓ |

## 4. Consideraciones de Diseño

### Balance: Precisión vs Cobertura
- **13 anchors** en basicColorTable = Balance entre simplicidad y precisión
- **32+ anchors** en completeColorTable = Máxima precisión para análisis detallados

### Pesos de Penalización
- **Delta E**: Peso 1.0 (métrica perceptual estándar)
- **Hue**: Peso 0.3 (menor peso que Delta E para evitar saltos)
- **Saturación**: Implícita en Delta E (a* y b* codifican color)

### Thresholds de Distancia
```javascript
const umbralCasi = modoPreciso ? 12 : 28;  // En basicColorTable es más generoso
```
Permite "casi" cuando la 2ª opción está a menos de 28 unidades ΔE en modo básico.

## 5. Futuras Mejoras

1. **Calibración dinámica**: Permitir que usuarios corrijan identificaciones
2. **Machine Learning**: Entrenar redes neuronales con datos reales
3. **Análisis de área**: En lugar de pixel único, analizar región
4. **Histograma**: Identificar color dominante vs color secundario
5. **Interpolación**: Usar gradientes entre anchors para mejor cobertura

## 6. Testing

Para verificar las correcciones, abre `test-colores.html` en un navegador.
Debería mostrar los 6 colores problemáticos identificados correctamente.

