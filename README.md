# LAB-4-EDA
import java.util.*;
import java.io.*;

class paciente {
    String nombre;
    String apellido;
    String id;
    int categoria;
    long tiempo_de_llegada;
    String estado;
    String area;
    Stack<String> historial_de_cambios;
    public paciente(String nombre, String apellido, String id, int categoria, long tiempo_de_llegada, String area) {
        this.nombre = nombre;
        this.apellido = apellido;
        this.id = id;
        this.categoria = categoria;
        this.tiempo_de_llegada = tiempo_de_llegada;
        this.estado = "en_espera";
        this.area = area;
        this.historial_de_cambios = new Stack<>();
    }
    public long tiempo_de_espera_actual(long tiempo_actual) {
        long tiempo_espera = (tiempo_actual - tiempo_de_llegada) / (60 * 1000);
        return tiempo_espera > 0 ? tiempo_espera : 0;
    }
    public void registrar_cambio(String descripcion) {
        historial_de_cambios.push(descripcion);
    }
    public String obtener_ultimo_cambio() {
        return historial_de_cambios.isEmpty() ? null : historial_de_cambios.pop();
    }
}
class area_de_atencion {
    String nombre;
    PriorityQueue<paciente> pacientes_heap;
    int capacidad_maxima;
    public area_de_atencion(String nombre, int capacidad_maxima) {
        this.nombre = nombre;
        this.capacidad_maxima = capacidad_maxima;
        this.pacientes_heap = new PriorityQueue<>(Comparator
                .comparingInt((paciente p) -> p.categoria)
                .thenComparingLong(p -> p.tiempo_de_espera_actual(System.currentTimeMillis())));
    }
    public void ingresar_paciente(paciente p) {
        if (pacientes_heap.size() < capacidad_maxima) {
            pacientes_heap.add(p);
            p.estado = "en_espera";
            p.registrar_cambio("Ingresado a área: " + nombre);
        }
    }
    public paciente atender_paciente() {
        paciente p = pacientes_heap.poll();
        if (p != null) {
            p.estado = "atendido";
            p.registrar_cambio("Atendido en área: " + nombre);
        }
        return p;
    }
    public boolean esta_saturada() {
        return pacientes_heap.size() >= capacidad_maxima;
    }
    public List<paciente> obtener_pacientes_por_heap_sort() {
        List<paciente> pacientes_ordenados = new ArrayList<>();
        PriorityQueue<paciente> copia = new PriorityQueue<>(pacientes_heap);
        while (!copia.isEmpty()) {
            pacientes_ordenados.add(copia.poll());
        }
        return pacientes_ordenados;
    }
}
class hospital {
    Map<String, paciente> pacientes_totales;
    PriorityQueue<paciente> cola_de_atencion;
    Map<String, area_de_atencion> areas_de_atencion;
    List<paciente> pacientes_atendidos;
    Map<Integer, Integer> pacientes_atendidos_por_categoria;
    Map<Integer, Long> tiempo_espera_acumulado;
    Map<Integer, Integer> pacientes_excedieron_tiempo;
    Map<Integer, Long> peores_tiempos_espera;
    public hospital() {
        this.pacientes_totales = new HashMap<>();
        this.cola_de_atencion = new PriorityQueue<>(Comparator
                .comparingInt((paciente p) -> p.categoria)
                .thenComparingLong(p -> p.tiempo_de_espera_actual(System.currentTimeMillis())));
        this.areas_de_atencion = new HashMap<>();
        this.pacientes_atendidos = new ArrayList<>();
        this.pacientes_atendidos_por_categoria = new HashMap<>();
        this.tiempo_espera_acumulado = new HashMap<>();
        this.pacientes_excedieron_tiempo = new HashMap<>();
        this.peores_tiempos_espera = new HashMap<>();
        for (int i = 1; i <= 5; i++) {
            pacientes_atendidos_por_categoria.put(i, 0);
            tiempo_espera_acumulado.put(i, 0L);
            pacientes_excedieron_tiempo.put(i, 0);
            peores_tiempos_espera.put(i, 0L);
        }
        areas_de_atencion.put("SAPU", new area_de_atencion("SAPU", 50));
        areas_de_atencion.put("urgencia_adulto", new area_de_atencion("urgencia_adulto", 100));
        areas_de_atencion.put("infantil", new area_de_atencion("infantil", 30));
    }
    public void registrar_paciente(paciente p) {
        pacientes_totales.put(p.id, p);
        cola_de_atencion.add(p);
        p.registrar_cambio("Registrado en el hospital");
    }
    public void reasignar_categoria(String id, int nueva_categoria) {
        paciente p = pacientes_totales.get(id);
        if (p != null) {
            p.registrar_cambio("Cambio de categoría de " + p.categoria + " a " + nueva_categoria);
            p.categoria = nueva_categoria;
            cola_de_atencion.remove(p);
            cola_de_atencion.add(p);
        }
    }
    public area_de_atencion obtenerArea(String nombre) {
    return areas_de_atencion.get(nombre);
}
    public paciente atender_siguiente(long tiempo_actual) {
        paciente p = cola_de_atencion.poll();
        if (p != null) {
            area_de_atencion area = areas_de_atencion.get(p.area);
            if (area != null && !area.esta_saturada()) {
                area.ingresar_paciente(p);
                pacientes_atendidos.add(p);
                long tiempo_espera = p.tiempo_de_espera_actual(tiempo_actual);
                pacientes_atendidos_por_categoria.put(p.categoria, pacientes_atendidos_por_categoria.get(p.categoria) + 1);
                tiempo_espera_acumulado.put(p.categoria, tiempo_espera_acumulado.get(p.categoria) + tiempo_espera);
                if (tiempo_espera > peores_tiempos_espera.get(p.categoria)) {
                    peores_tiempos_espera.put(p.categoria, tiempo_espera);
                }
                if (excedio_tiempo_maximo(p, tiempo_actual)) {
                    pacientes_excedieron_tiempo.put(p.categoria, pacientes_excedieron_tiempo.get(p.categoria) + 1);
                }
                return p;
            }
        }
        return null;
    }
    private boolean excedio_tiempo_maximo(paciente p, long tiempo_actual) {
        long tiempo_espera = p.tiempo_de_espera_actual(tiempo_actual);
        switch (p.categoria) {
            case 1: return tiempo_espera > 0;
            case 2: return tiempo_espera > 30;
            case 3: return tiempo_espera > 90;
            case 4: return tiempo_espera > 180;
            default: return false;
        }
    }
    public List<paciente> obtener_pacientes_por_categoria(int categoria) {
        List<paciente> pacientes = new ArrayList<>();
        for (paciente p : cola_de_atencion) {
            if (p.categoria == categoria) {
                pacientes.add(p);
            }
        }
        return pacientes;
    }
    public void verificar_esperas_largas(long tiempo_actual) {
        for (paciente p : cola_de_atencion) {
            if (p.tiempo_de_espera_actual(tiempo_actual) > 180 && p.categoria > 1) {
                reasignar_categoria(p.id, p.categoria - 1);
            }
        }
    }
}

class generador_pacientes {
    private static final String[] NOMBRES = {"Juan", "Maria", "Pedro", "Ana", "Luis", "Carmen", "Jose", "Isabel"};
    private static final String[] APELLIDOS = {"Garcia", "Rodriguez", "Gonzalez", "Fernandez", "Lopez", "Martinez", "Sanchez", "Perez"};
    private int contador_id = 1;
    private String generar_rut() {
        Random random = new Random();
        int num = 1000000 + random.nextInt(9000000);
        int dv = random.nextInt(10);
        return num + "-" + (dv == 10 ? "k" : dv);
    }
    public paciente generar_paciente(long tiempo_llegada) {
        Random random = new Random();
        String nombre = NOMBRES[random.nextInt(NOMBRES.length)];
        String apellido = APELLIDOS[random.nextInt(APELLIDOS.length)];
        String id = generar_rut();
        int categoria;
        double prob = random.nextDouble();
        if (prob < 0.10) categoria = 1;
        else if (prob < 0.25) categoria = 2;
        else if (prob < 0.43) categoria = 3;
        else if (prob < 0.70) categoria = 4;
        else categoria = 5;
        String area;
        if (categoria <= 2) area = "urgencia_adulto";
        else if (random.nextDouble() < 0.5) area = "SAPU";
        else area = "infantil";
        return new paciente(nombre, apellido, id, categoria, tiempo_llegada, area);
    }
    public List<paciente> generar_lista_pacientes(int cantidad, long timestamp_inicio) {
        List<paciente> pacientes = new ArrayList<>();
        for (int i = 0; i < cantidad; i++) {
            long tiempo_llegada = timestamp_inicio + (i * 10 * 60 * 1000);
            pacientes.add(generar_paciente(tiempo_llegada));
        }
        return pacientes;
    }
    public void guardar_pacientes_en_archivo(List<paciente> pacientes, String nombre_archivo) throws IOException {
        try (PrintWriter writer = new PrintWriter(new FileWriter(nombre_archivo))) {
            for (paciente p : pacientes) {
                writer.println(p.id + "," + p.nombre + "," + p.apellido + "," + p.categoria + "," + p.tiempo_de_llegada + "," + p.area);
            }
        }
    }
}
public void simular(int pacientesPorDia) {
    public static void main(String[] args) throws IOException {
        long tiempo_inicio = System.currentTimeMillis();
        generador_pacientes generador = new generador_pacientes();
        List<paciente> pacientes = generador.generar_lista_pacientes(200, tiempo_inicio);
        generador.guardar_pacientes_en_archivo(pacientes, "Pacientes_24h.txt");
        hospital hospital = new hospital();
        int contador_nuevos = 0;
        for (int minuto = 0; minuto < 1440; minuto++) {
            long tiempo_actual = tiempo_inicio + (minuto * 60 * 1000);
            if (minuto % 10 == 0 && minuto/10 < pacientes.size()) {
                hospital.registrar_paciente(pacientes.get(minuto/10));
                contador_nuevos++;
                if (contador_nuevos >= 3) {
                    hospital.atender_siguiente(tiempo_actual);
                    hospital.atender_siguiente(tiempo_actual);
                    contador_nuevos = 0;
                }
            }
            if (minuto % 15 == 0) {
                hospital.atender_siguiente(tiempo_actual);
            }
            hospital.verificar_esperas_largas(tiempo_actual);
            if (minuto == 720) {
                paciente pacienteC4 = pacientes.stream()
                        .filter(p -> p.categoria == 4)
                        .findFirst()
                        .orElse(null);
                if (pacienteC4 != null) {
                    hospital.reasignar_categoria(pacienteC4.id, 3);
                }
            }
            if (minuto % 60 == 0) {
                System.out.println("=== Reporte hora " + (minuto/60) + " ===");
                for (int cat = 1; cat <= 5; cat++) {
                    System.out.printf("C%d: Atendidos=%d | Espera promedio=%.1f min | Excedidos=%d | Peor espera=%d%n",
                            cat,
                            hospital.pacientes_atendidos_por_categoria.get(cat),
                            hospital.tiempo_espera_acumulado.get(cat) / (double) Math.max(1, hospital.pacientes_atendidos_por_categoria.get(cat)),
                            hospital.pacientes_excedieron_tiempo.get(cat),
                            hospital.peores_tiempos_espera.get(cat));
                }
            }
        }
    }
}
