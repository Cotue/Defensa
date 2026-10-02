# Estudio para la defensa — Enfoque en código y manejo de datos

La mejor forma de estudiar este proyecto es seguir **qué función llama a cuál, qué struct recibe, qué campos modifica y cómo un dato entra desde un archivo y termina en el JSON**.

Voy a usar fragmentos del código real que subiste y voy a indicar de qué archivo salen.

---

# 1. Punto de entrada: `main.c`

Todo el proyecto se conecta aquí.

```c
// main.c

int main(int argc, char *argv[]) {
    if (argc != 4) {
        printf("Uso: %s <catalogo.tsv> <historial.txt> <salida.json>\n", argv[0]);
        return ERROR_ARGS;
    }

    Catalog catalog;
    catalog_init(&catalog);

    StudentHistory history;
    student_history_init(&history);
```

El programa espera:

```text
argv[1] → catálogo TSV
argv[2] → historial TXT
argv[3] → archivo JSON de salida
```

Luego crea dos estructuras locales:

```c
Catalog catalog;
StudentHistory history;
```

y las inicializa antes de utilizarlas. main

Después ocurre el flujo principal:

```c
// main.c

Status result = parse_catalog(argv[1], &catalog);

if (result == SUCCESS) {

    result = parse_student_history(
        argv[2],
        &catalog,
        &history
    );

    if (result == SUCCESS) {

        detect_catalog_conflicts(&catalog);

        evaluate_catalog_eligibility(
            &catalog,
            &history
        );

        result = serialize_catalog_to_json(
            &catalog,
            argv[3]
        );
    }
}
```

main

Este fragmento es probablemente **el más importante de toda la defensa**.

Mentalmente:

```text
parse_catalog()
      ↓
llena Catalog

parse_student_history()
      ↓
llena StudentHistory

detect_catalog_conflicts()
      ↓
modifica Groups y Courses

evaluate_catalog_eligibility()
      ↓
modifica Courses

serialize_catalog_to_json()
      ↓
lee Catalog y escribe JSON
```

---

# 2. ¿Qué contiene realmente `Catalog`?

El `.h` solo necesitas conocerlo resumidamente:

```c
typedef struct {
    char *career_code;
    char *career_name;

    Course *courses;
    size_t course_count;
    size_t course_capacity;
} Catalog;
```

catalog

Lo importante es interpretar esto:

```text
courses
→ arreglo dinámico de Course

course_count
→ cantidad de cursos realmente almacenados

course_capacity
→ cantidad de espacios reservados
```

Ejemplo:

```text
Catalog
│
├── course_count = 3
├── course_capacity = 8
│
└── courses
    ├── CE1101
    ├── MA1102
    ├── FI1101
    ├── libre
    ├── libre
    └── ...
```

---

# 3. Cómo comienza vacío un `Catalog`

Esto sale de `catalog.c`:

```c
// catalog.c

void catalog_init(Catalog *catalog)
{
    if (catalog == NULL) {
        return;
    }

    catalog->career_code = NULL;
    catalog->career_name = NULL;

    catalog->courses = NULL;
    catalog->course_count = 0;
    catalog->course_capacity = 0;
}
```

catalog

Cuando en `main` hacemos:

```c
catalog_init(&catalog);
```

el estado queda:

```text
career_code = NULL
career_name = NULL

courses = NULL
course_count = 0
course_capacity = 0
```

Todavía **no hay memoria reservada para cursos**.

La memoria aparece cuando realmente agregamos el primer curso.

---

# 4. Cómo `catalog_parser.c` lee una línea

En `parse_catalog()` existe:

```c
char line[MAX_LINE_LENGTH];
```

y luego:

```c
while (fgets(line, sizeof(line), file)) {
```

catalog_parser

`fgets()` lee una línea completa del TSV y la coloca dentro de `line`.

Ejemplo:

```text
IC<TAB>1<TAB>CE1101<TAB>Programación<TAB>4...
```

Después elimina:

```text
\r
\n
```

con:

```c
line[strcspn(line, "\r\n")] = '\0';
```

catalog_parser

---

# 5. Cómo una línea se divide en columnas

Este bloque es importante:

```c
// catalog_parser.c

char *columns[12] = {0};
int col_count = 0;
char *start = line;

while (start) {

    if (col_count < 12)
        columns[col_count] = start;

    col_count++;

    char *tab = strchr(start, '\t');

    if (tab) {
        *tab = '\0';
        start = tab + 1;
    } else {
        start = NULL;
    }
}
```

catalog_parser

Supongamos:

```text
IC<TAB>1<TAB>CE1101<TAB>Programación
```

Primero:

```text
start
 ↓
"IC<TAB>1<TAB>CE1101..."
```

`strchr()` encuentra el primer tab.

Entonces:

```c
*tab = '\0';
```

transforma temporalmente:

```text
IC\01<TAB>CE1101...
```

Ahora:

```c
columns[0]
```

apunta a:

```text
"IC"
```

Después:

```c
start = tab + 1;
```

avanza al siguiente campo.

Resultado final:

```text
columns[0] = "IC"
columns[1] = "1"
columns[2] = "CE1101"
columns[3] = "Programación"
...
```

---

# 6. Cómo se sacan los datos de `columns`

El parser después simplemente asigna punteros:

```c
const char *career = columns[0];
const char *semester_str = columns[1];
const char *code = columns[2];
const char *name = columns[3];
const char *credits_str = columns[4];
const char *req_str = columns[5];
const char *coreq_str = columns[6];
const char *group_str = columns[7];
const char *day_str = columns[8];
const char *start_str = columns[9];
const char *end_str = columns[10];
```

catalog_parser

Aquí todavía casi todo son:

```c
const char *
```

o sea, strings.

---

# 7. Cómo se validan los datos antes de convertirlos

Antes de usar `atoi()` o convertir horarios, se llaman funciones del `validator.c`:

```c
if (validate_course_identity(code, name) != SUCCESS ||
    validate_day_format(day_str) != SUCCESS ||
    validate_time_format(start_str, end_str) != SUCCESS ||
    validate_semester_format(semester_str) != SUCCESS ||
    validate_credits_format(credits_str) != SUCCESS ||
    validate_group_format(group_str) != SUCCESS) {

    ...
    return ERROR_INVALID_FORMAT;
}
```

catalog_parser

Esto es importante porque no se hace:

```c
atoi("hola");
```

sin verificar primero que realmente sea un número válido.

---

# 8. Ejemplo: validar un grupo

En `validator.c`:

```c
Status validate_group_format(const char *group_str) {

    if (!group_str || group_str[0] == '\0') {
        return ERROR_INVALID_FORMAT;
    }

    for (int i = 0; group_str[i] != '\0'; i++) {

        if (!isdigit(group_str[i])) {
            return ERROR_INVALID_FORMAT;
        }
    }

    int group = atoi(group_str);

    if (group <= 0) {
        return ERROR_INVALID_FORMAT;
    }

    return SUCCESS;
}
```

validator

Ejemplo:

```text
"12"
```

Se revisa:

```text
'1' → dígito
'2' → dígito
```

Luego:

```c
atoi("12")
```

produce:

```text
12
```

---

# 9. Cómo se convierte un horario

Primero se valida que tenga formato:

```text
HH:MM
```

Después el parser hace:

```c
int sh, sm, eh, em;

sscanf(start_str, "%d:%d", &sh, &sm);
sscanf(end_str, "%d:%d", &eh, &em);

int start_minutes = sh * 60 + sm;
int end_minutes = eh * 60 + em;
```

catalog_parser

Ejemplo:

```text
07:30
```

produce:

```text
sh = 7
sm = 30
```

Entonces:

```text
7 * 60 + 30
= 450
```

---

# 10. Cómo se representa el horario en memoria

El struct es:

```c
typedef struct {
    Day day;
    int start_minutes;
    int end_minutes;
} Schedule;
```

schedule

Entonces:

```text
LUN 07:30 - 09:20
```

se convierte en:

```c
Schedule {
    day = MONDAY,
    start_minutes = 450,
    end_minutes = 560
}
```

El motivo es que después resulta mucho más sencillo hacer:

```c
450 < 620
```

que comparar horas en forma textual.

---

# 11. Cómo se crea un curso

El parser pregunta:

```c
Course *course =
    find_course_by_code(catalog, code);
```

catalog_parser

Esta función sale de `catalog.c`:

```c
Course* find_course_by_code(
    Catalog *catalog,
    const char *code
) {
    for (size_t i = 0;
         i < catalog->course_count;
         i++) {

        if (strcmp(
                catalog->courses[i].code,
                code
            ) == 0) {

            return &catalog->courses[i];
        }
    }

    return NULL;
}
```

catalog

Si devuelve:

```c
NULL
```

significa:

```text
ese curso todavía no existe
```

Entonces:

```c
course = catalog_add_course(
    catalog,
    code,
    name,
    credits,
    semester
);
```

---

# 12. Cómo se crea dinámicamente la lista de cursos

En `catalog.c`:

```c
if (catalog->course_count ==
    catalog->course_capacity) {

    size_t new_cap =
        (catalog->course_capacity == 0)
        ? INITIAL_CAPACITY
        : catalog->course_capacity * 2;

    Course *temp =
        realloc(
            catalog->courses,
            new_cap * sizeof(Course)
        );

    if (!temp)
        return NULL;

    catalog->courses = temp;
    catalog->course_capacity = new_cap;
}
```

catalog

Al principio:

```text
count = 0
capacity = 0
courses = NULL
```

Cuando llega el primer curso:

```text
capacity = 8
```

Si eventualmente se ocupan los 8:

```text
8 → 16
```

Luego:

```text
16 → 32
```

---

# 13. Cómo se inicializa el nuevo `Course`

Después del `realloc()`:

```c
Course *new_course =
    &catalog->courses[
        catalog->course_count
    ];

course_init(new_course);
```

catalog

`course_init()` pone inicialmente:

```c
course->code = NULL;
course->name = NULL;

course->credits = 0;
course->semester = 0;

code_list_init(&course->requirements);
code_list_init(&course->corequisites);
code_list_init(&course->pending_corequisites);

course->groups = NULL;
course->group_count = 0;
course->group_capacity = 0;

course->has_conflict = false;
course->can_enroll = false;
```

course

---

# 14. Cómo se almacenan `code` y `name`

Luego:

```c
new_course->code =
    malloc(strlen(code) + 1);

new_course->name =
    malloc(strlen(name) + 1);
```

Después:

```c
strcpy(new_course->code, code);
strcpy(new_course->name, name);
```

catalog

Esto es fundamental.

El parser está usando un buffer:

```c
char line[MAX_LINE_LENGTH];
```

Ese buffer se vuelve a utilizar en la próxima línea.

Entonces el `Course` necesita **su propia copia** de los strings.

No sería seguro hacer:

```c
new_course->code = code;
```

porque `code` apunta dentro del buffer temporal del parser.

---

# 15. Cómo se crea una lista de requisitos

Supongamos que en el TSV tenemos:

```text
CE1101;MA1102
```

El parser llama:

```c
parse_code_list(
    &course->requirements,
    req_str
);
```

catalog_parser

Dentro de `parse_code_list()`:

```c
char *delim =
    strstr(start, LIST_DELIMITER);

if (delim)
    *delim = '\0';
```

Después:

```c
code_list_add(list, start);
```

catalog_parser

Entonces:

```text
"CE1101;MA1102"
```

primero queda temporalmente:

```text
"CE1101\0MA1102"
```

Se agrega:

```text
CE1101
```

Después `start` avanza y agrega:

```text
MA1102
```

Resultado:

```text
requirements
├── items[0] = "CE1101"
└── items[1] = "MA1102"

count = 2
```

---

# 16. Cómo funciona realmente `CodeList`

Código de `code_list.c`:

```c
int code_list_add(
    CodeList *list,
    const char *code
) {
    if (list->count == list->capacity) {

        size_t new_cap =
            (list->capacity == 0)
            ? INITIAL_CAPACITY
            : list->capacity * 2;

        char **temp =
            realloc(
                list->items,
                new_cap * sizeof(char *)
            );

        if (!temp)
            return 0;

        list->items = temp;
        list->capacity = new_cap;
    }
```

Después crea una copia del string:

```c
list->items[list->count] =
    malloc(strlen(code) + 1);

if (!list->items[list->count])
    return 0;

strcpy(
    list->items[list->count],
    code
);

list->count++;
```

code_list

Mentalmente:

```text
CodeList
│
├── items
│   ├── "CE1101"
│   ├── "MA1102"
│   └── ...
│
├── count
└── capacity
```

---

# 17. Cómo se crea el grupo

Después de tener el curso:

```c
Group *group =
    find_group_by_number(
        course,
        group_number
    );
```

catalog_parser

Si no existe:

```c
group =
    course_add_group(
        course,
        group_number
    );
```

Dentro de `course.c`:

```c
if (course->group_count ==
    course->group_capacity) {

    size_t new_cap =
        (course->group_capacity == 0)
        ? INITIAL_CAPACITY
        : course->group_capacity * 2;

    Group *temp =
        realloc(
            course->groups,
            new_cap * sizeof(Group)
        );
```

course

Luego:

```c
Group *new_group =
    &course->groups[
        course->group_count
    ];

group_init(new_group);
new_group->number = number;

course->group_count++;
```

course

---

# 18. Cómo se agrega un horario al grupo

Desde `catalog_parser.c`:

```c
group_add_schedule(
    group,
    day,
    start_minutes,
    end_minutes
);
```

catalog_parser

Dentro de `group.c`:

```c
Schedule *new_sched =
    &group->schedules[
        group->schedule_count
    ];

schedule_init(new_sched);

new_sched->day = day;
new_sched->start_minutes =
    start_minutes;

new_sched->end_minutes =
    end_minutes;

group->schedule_count++;
```

group

Entonces una fila como:

```text
CE1101 G1 LUN 07:30 09:20
```

produce:

```text
Course CE1101
└── Group 1
    └── Schedule
        ├── MONDAY
        ├── 450
        └── 560
```

Y otra fila:

```text
CE1101 G1 JUE 07:30 09:20
```

no crea otro curso ni otro grupo:

```text
Course CE1101
└── Group 1
    ├── MONDAY 450-560
    └── THURSDAY 450-560
```

---

# 19. Cómo se lee el historial

En `history_parser.c`:

```c
char line[MAX_LINE_LENGTH];

while (fgets(
        line,
        sizeof(line),
        file
    ) != NULL) {
```

history_parser

Supongamos:

```text
    CE1101
```

La función:

```c
trim_whitespace()
```

hace:

```c
while (isspace(
        (unsigned char)*text
    )) {

    text++;
}
```

history_parser

Entonces el puntero simplemente avanza:

```text
"    CE1101"
 ↑
 text

↓ ↓ ↓ ↓

"    CE1101"
     ↑
     text
```

Ahora apunta directamente a:

```text
CE1101
```

---

# 20. Cómo elimina espacios al final

Después:

```c
char *end =
    text + strlen(text) - 1;

while (end > text &&
       isspace(
           (unsigned char)*end
       )) {

    end--;
}

end[1] = '\0';
```

history_parser

Si teníamos:

```text
"CE1101   "
```

coloca:

```text
'\0'
```

después del último carácter válido:

```text
C E 1 1 0 1 \0
```

---

# 21. Cómo se evita un curso repetido en el historial

Después del trim:

```c
if (code_list_contains(
        &history->approved_courses,
        code
    )) {

    ...
    return ERROR_INVALID_FORMAT;
}
```

history_parser

`code_list_contains()` hace:

```c
for (size_t i = 0;
     i < list->count;
     i++) {

    if (strcmp(
            list->items[i],
            code
        ) == 0) {

        return 1;
    }
}

return 0;
```

code_list

---

# 22. Cómo se comprueba que el curso exista

Después:

```c
if (find_course_by_code(
        catalog,
        code
    ) == NULL) {

    return ERROR_INVALID_FORMAT;
}
```

history_parser

Entonces el flujo es:

```text
leer "CE1101"
      ↓
trim
      ↓
¿duplicado?
      ↓ no
¿existe en Catalog?
      ↓ sí
code_list_add()
```

---

# 23. Cómo se almacena el historial

Finalmente:

```c
code_list_add(
    &history->approved_courses,
    code
);
```

history_parser

Entonces:

```text
StudentHistory
└── approved_courses
    ├── "CE1101"
    ├── "MA1102"
    └── "FI1101"
```

`StudentHistory` internamente es solamente:

```c
typedef struct {
    CodeList approved_courses;
} StudentHistory;
```

student_history

---

# 24. Cómo funciona la elegibilidad en código

Desde `main.c`:

```c
evaluate_catalog_eligibility(
    &catalog,
    &history
);
```

La función:

```c
for (int i = 0;
     i < (int)catalog->course_count;
     i++) {

    evaluate_course_eligibility(
        &catalog->courses[i],
        history
    );
}
```

eligibility

Es decir:

```text
Course 0 → evaluar
Course 1 → evaluar
Course 2 → evaluar
...
```

---

# 25. Cómo comprueba si el curso ya estaba aprobado

```c
if (code_list_contains(
        &history->approved_courses,
        course->code
    )) {

    course->can_enroll = false;
    return;
}
```

eligibility

Aquí se reutiliza exactamente el mismo `CodeList`.

Por ejemplo:

```text
course->code
= "CE1101"

approved_courses
├── "CE1101"
├── "MA1102"
└── ...
```

`code_list_contains()` devuelve:

```text
1
```

Entonces:

```c
course->can_enroll = false;
```

---

# 26. Cómo revisa los requisitos

```c
for (int i = 0;
     i < (int)course->requirements.count;
     i++) {

    const char *req_code =
        course->requirements.items[i];

    if (code_list_contains(
            &history->approved_courses,
            req_code
        ) == 0) {

        course->can_enroll = false;
        break;
    }
}
```

eligibility

Ejemplo:

```text
requirements
├── CE1101
└── MA1102
```

Historial:

```text
approved_courses
└── CE1101
```

Recorrido:

```text
i = 0
req_code = CE1101
→ existe

i = 1
req_code = MA1102
→ no existe
→ can_enroll = false
→ break
```

---

# 27. Cómo se crean los correquisitos pendientes

Primero:

```c
code_list_free(
    &course->pending_corequisites
);

code_list_init(
    &course->pending_corequisites
);
```

Después:

```c
for (int i = 0;
     i < (int)course->corequisites.count;
     i++) {

    const char *coreq_code =
        course->corequisites.items[i];

    if (code_list_contains(
            &history->approved_courses,
            coreq_code
        ) == 0) {

        code_list_add(
            &course->pending_corequisites,
            coreq_code
        );
    }
}
```

eligibility

Supongamos:

```text
corequisites
├── MA1102
└── FI1101
```

Historial:

```text
approved_courses
└── FI1101
```

Entonces:

```text
MA1102 → no aprobado → pending
FI1101 → aprobado → no se agrega
```

Resultado:

```text
pending_corequisites
└── MA1102
```

---

# 28. Cómo comienza la detección de choques

Desde `main.c`:

```c
detect_catalog_conflicts(&catalog);
```

Primero limpia resultados anteriores:

```c
clear_catalog_conflicts(catalog);
```

Dentro:

```c
course->has_conflict = false;

for (size_t j = 0;
     j < course->group_count;
     j++) {

    group_clear_conflicts(
        &course->groups[j]
    );
}
```

conflict_detector

Esto es porque los conflictos son **datos calculados**.

---

# 29. Comparación `Schedule` contra `Schedule`

Esta función es pequeña pero importantísima:

```c
static int schedules_conflict(
    const Schedule *schedule_a,
    const Schedule *schedule_b
)
{
    if (schedule_a->day !=
        schedule_b->day) {

        return 0;
    }

    return (
        schedule_a->start_minutes <
            schedule_b->end_minutes
        &&
        schedule_b->start_minutes <
            schedule_a->end_minutes
    );
}
```

conflict_detector

Ejemplo:

```text
A = 450 - 560
B = 510 - 620
```

Prueba:

```text
450 < 620 → true
510 < 560 → true
```

Entonces:

```text
choque
```

---

# 30. Comparación `Group` contra `Group`

Como un grupo tiene varios schedules:

```c
for (size_t i = 0;
     i < group_a->schedule_count;
     i++) {

    for (size_t j = 0;
         j < group_b->schedule_count;
         j++) {

        if (schedules_conflict(
                &group_a->schedules[i],
                &group_b->schedules[j]
            )) {

            return 1;
        }
    }
}
```

conflict_detector

Ejemplo:

```text
Group A
├── LUN
└── JUE

Group B
├── MAR
└── JUE
```

Se comparan:

```text
A.LUN vs B.MAR
A.LUN vs B.JUE
A.JUE vs B.MAR
A.JUE vs B.JUE
```

Si una sola pareja choca:

```c
return 1;
```

---

# 31. Cómo se evitan comparaciones duplicadas

El detector hace:

```c
for (size_t i = 0;
     i < catalog->course_count;
     i++) {

    for (size_t j = i + 1;
         j < catalog->course_count;
         j++) {
```

conflict_detector

Si tenemos:

```text
A
B
C
```

compara:

```text
A-B
A-C
B-C
```

pero no:

```text
A-A
B-B
B-A
C-A
...
```

---

# 32. Cómo se guarda un choque

Cuando encuentra conflicto:

```c
group_add_conflict(
    group_a,
    course_b->code,
    group_b->number
);
```

Después también:

```c
group_add_conflict(
    group_b,
    course_a->code,
    group_a->number
);
```

conflict_detector

Por eso la relación queda simétrica.

---

# 33. Cómo `group_add_conflict()` almacena datos

Dentro de `group.c`:

```c
char *code_copy =
    malloc(strlen(course_code) + 1);

strcpy(
    code_copy,
    course_code
);
```

Después:

```c
GroupConflict *new_conflict =
    &group->conflicts[
        group->conflict_count
    ];

new_conflict->course_code =
    code_copy;

new_conflict->group_number =
    group_number;

group->conflict_count++;
group->has_conflict = true;
```

group

Ejemplo:

```text
CE1101 G1
choca con
MA1102 G4
```

En `CE1101 G1` se almacena:

```text
GroupConflict
├── course_code = "MA1102"
└── group_number = 4
```

---

# 34. Cómo se escribe finalmente el JSON

En `serializer.c`:

```c
FILE *file =
    fopen(filepath, "w");
```

serializer

La `"w"` significa:

```text
abrir archivo para escritura
```

Después:

```c
fprintf(file, "{\n");
fprintf(
    file,
    "  \"schemaVersion\": \"%s\",\n",
    OUTPUT_SCHEMA_VERSION
);
```

serializer

---

# 35. Cómo se recorre `Catalog` para escribir cursos

```c
for (size_t i = 0;
     i < catalog->course_count;
     i++) {

    Course *c =
        &catalog->courses[i];
```

Luego:

```c
fprintf(
    file,
    "\"codigo\": \"%s\"",
    c->code
);

fprintf(
    file,
    "\"creditos\": %d",
    c->credits
);
```

serializer

Aquí tienes manejo de tipos:

```text
char * → %s
int    → %d
```

---

# 36. Cómo se escribe una lista JSON

`requirements`, `corequisites` y `pending_corequisites` se escriben mediante:

```c
static void print_code_list_json(
    FILE *file,
    const CodeList *list
) {
    fprintf(file, "[");

    for (size_t i = 0;
         i < list->count;
         i++) {

        fprintf(
            file,
            "\"%s\"",
            list->items[i]
        );

        if (i < list->count - 1) {
            fprintf(file, ", ");
        }
    }

    fprintf(file, "]");
}
```

serializer

Si:

```text
items[0] = CE1101
items[1] = MA1102
```

produce:

```json
["CE1101", "MA1102"]
```

---

# 37. Cómo se escribe cada horario

El serializer recorre:

```c
for (size_t k = 0;
     k < g->schedule_count;
     k++) {

    Schedule *s =
        &g->schedules[k];
```

Después:

```c
fprintf(
    file,
    "\"dia\": \"%s\"",
    day_to_string(s->day)
);
```

y:

```c
print_time_json(
    file,
    s->start_minutes
);
```

serializer

---

# 38. Cómo convierte minutos nuevamente a hora

```c
static void print_time_json(
    FILE *file,
    int minutes
) {
    int h = minutes / 60;
    int m = minutes % 60;

    fprintf(
        file,
        "\"%02d:%02d\"",
        h,
        m
    );
}
```

serializer

Si recibe:

```text
570
```

entonces:

```text
570 / 60 = 9
570 % 60 = 30
```

produce:

```json
"09:30"
```

---

# 39. Flujo de un dato completo

Supongamos esta fila:

```text
IC  2  CE2201  Estructuras  4  CE1101  MA1102  1  LUN  09:30  11:20
```

## Entrada

```text
"2"
→ atoi()
→ semester = 2

"4"
→ atoi()
→ credits = 4

"LUN"
→ parse_day_string()
→ MONDAY

"09:30"
→ 570

"11:20"
→ 680
```

## Modelo

```text
Course CE2201
│
├── requirements
│   └── CE1101
│
├── corequisites
│   └── MA1102
│
└── Group 1
    └── Schedule
        ├── MONDAY
        ├── 570
        └── 680
```

## Historial

Supongamos:

```text
CE1101
```

Entonces:

```text
CE1101 requisito
→ aprobado

MA1102 correquisito
→ no aprobado
→ pending_corequisites
```

## Salida

```json
{
  "codigo": "CE2201",
  "semestre": 2,
  "requisitos": ["CE1101"],
  "correquisitos": ["MA1102"],
  "correquisitosPendientes": ["MA1102"],
  "grupos": [
    {
      "numero": 1,
      "horarios": [
        {
          "dia": "LUN",
          "inicio": "09:30",
          "fin": "11:20"
        }
      ]
    }
  ]
}
```

Ese ejemplo es muy bueno para explicar el proyecto completo en la defensa.

---

# 40. Memoria — qué se libera y desde dónde

El final de `main.c`:

```c
student_history_free(&history);
catalog_free(&catalog);
```

main

`catalog_free()`:

```c
for (size_t i = 0;
     i < catalog->course_count;
     i++) {

    course_free(
        &catalog->courses[i]
    );
}

free(catalog->courses);
```

catalog

`course_free()`:

```c
free(course->code);
free(course->name);

code_list_free(
    &course->requirements
);

code_list_free(
    &course->corequisites
);

code_list_free(
    &course->pending_corequisites
);
```

Después:

```c
for (...) {
    group_free(
        &course->groups[i]
    );
}

free(course->groups);
```

course

La liberación es:

```text
Catalog
 ↓
Course
 ↓
Group
 ↓
Schedule / GroupConflict
```

---

# 41. Preguntas de defensa — conceptos + código

## «Explíqueme cómo pasa una fila del TSV hasta convertirse en un `Schedule`.»

Primero `fgets()` carga una línea. Después `catalog_parser.c` la divide por `\t` en `columns[]`. Los datos numéricos se validan y se convierten. `parse_day_string()` transforma `"LUN"` a `MONDAY`, y las horas se convierten a minutos. Después se busca o crea el `Course`, se busca o crea el `Group` y finalmente `group_add_schedule()` crea el `Schedule`.

Código clave:

```c
Day day =
    parse_day_string(day_str);

int start_minutes =
    sh * 60 + sm;

Group *group =
    find_group_by_number(
        course,
        group_number
    );

group_add_schedule(
    group,
    day,
    start_minutes,
    end_minutes
);
```

---

## «¿Por qué utilizan `count` y `capacity`?»

Porque los arreglos son dinámicos.

Ejemplo:

```text
count = 3
capacity = 8
```

Hay tres elementos reales, pero memoria reservada para ocho.

Cuando:

```c
count == capacity
```

se hace:

```c
realloc()
```

y generalmente la capacidad se duplica.

---

## «Muéstreme dónde crecen dinámicamente los cursos.»

En `catalog.c`:

```c
if (catalog->course_count ==
    catalog->course_capacity) {

    Course *temp =
        realloc(
            catalog->courses,
            new_cap * sizeof(Course)
        );
}
```

catalog

El mismo patrón aparece en grupos, horarios, conflictos y `CodeList`.

---

## «¿Por qué usan un puntero temporal con `realloc()`?»

Porque `realloc()` puede fallar.

```c
Course *temp =
    realloc(...);

if (!temp)
    return NULL;

catalog->courses = temp;
```

Si asignáramos directamente el resultado a `catalog->courses`, un fallo podría hacer que perdiéramos la referencia a la memoria anterior.

---

## «¿Qué diferencia hay entre `malloc()` y `realloc()` en este proyecto?»

`malloc()` se usa normalmente para crear memoria nueva:

```c
new_course->code =
    malloc(strlen(code) + 1);
```

`realloc()` se utiliza para ampliar arreglos dinámicos:

```c
realloc(
    catalog->courses,
    new_cap * sizeof(Course)
);
```

---

## «¿Por qué hacen `strlen(code) + 1`?»

Por el carácter final:

```text
'\0'
```

Un string C como:

```text
CE1101
```

realmente necesita:

```text
C E 1 1 0 1 \0
```

---

## «¿Por qué copian los códigos y no guardan directamente el puntero?»

Porque muchos punteros recibidos vienen de buffers temporales del parser.

El `Course` y `CodeList` necesitan poseer memoria independiente:

```c
malloc(...)
strcpy(...)
```

De esa forma, cuando el parser reutiliza `line`, el dato almacenado no cambia.

---

## «¿Cómo detectan que un requisito está aprobado?»

La función toma:

```c
course->requirements.items[i]
```

y lo busca dentro de:

```c
history->approved_courses
```

mediante:

```c
code_list_contains()
```

Si alguno no existe, se marca:

```c
course->can_enroll = false;
```

---

## «¿Qué diferencia hay entre requisito y correquisito en el código?»

Los requisitos se utilizan para decidir:

```c
can_enroll
```

Los correquisitos faltantes se agregan a:

```c
pending_corequisites
```

mediante:

```c
code_list_add(
    &course->pending_corequisites,
    coreq_code
);
```

---

## «¿Por qué un `Group` tiene varios `Schedule`?»

Porque un grupo puede impartirse varios días.

Ejemplo:

```text
Grupo 1
├── LUN 07:30
└── JUE 07:30
```

Por eso existe:

```c
Schedule *schedules;
size_t schedule_count;
size_t schedule_capacity;
```

---

## «¿Cómo detectan exactamente un choque?»

Primero los días deben coincidir.

Después:

```c
startA < endB &&
startB < endA
```

Esto detecta superposición.

Por eso:

```text
07:30-09:20
09:20-11:10
```

no choca, porque solo tocan el mismo instante.

---

## «¿Por qué los choques se guardan en ambos grupos?»

Porque la relación es simétrica.

Si:

```text
CE1101 G1
```

choca con:

```text
MA1102 G2
```

se registra:

```text
CE1101 G1 → MA1102 G2
MA1102 G2 → CE1101 G1
```

El código hace dos llamadas consecutivas a `group_add_conflict()`. conflict_detector

---

## «¿Por qué `GroupConflict` no guarda un `Group *`?»

Porque los grupos están dentro de arreglos que pueden cambiar de dirección después de un `realloc()`.

Se guarda una identidad más estable:

```text
course_code
+
group_number
```

---

## «¿Qué diferencia existe entre `Group.has_conflict` y `Course.has_conflict`?»

`Group.has_conflict`:

```text
ese grupo concreto tiene al menos un choque
```

`Course.has_conflict`:

```text
al menos uno de los grupos del curso tiene choque
```

---

## «¿Un choque hace que `can_enroll` sea falso?»

No.

Son dos conceptos separados:

```text
has_conflict
→ horarios

can_enroll
→ situación académica
```

---

## «¿Qué hace exactamente el serializer?»

No calcula reglas nuevas.

Recorre:

```text
Catalog
 ↓
Course[]
 ↓
Group[]
 ↓
Schedule[]
 ↓
GroupConflict[]
```

y escribe cada valor mediante `fprintf()`.

---

## «¿Cómo escriben un booleano en JSON?»

Por ejemplo:

```c
fprintf(
    file,
    "\"tieneChoque\": %s",
    g->has_conflict
        ? "true"
        : "false"
);
```

serializer

El operador ternario:

```c
condicion ? valor_si_true : valor_si_false
```

devuelve el texto correcto.

---

## «¿Cómo escriben un arreglo JSON?»

Para `CodeList`:

```c
fprintf(file, "[");

for (...) {
    fprintf(
        file,
        "\"%s\"",
        list->items[i]
    );
}

fprintf(file, "]");
```

serializer

---

## «¿Qué significa `&catalog->courses[i]`?»

`catalog->courses[i]` es un `Course`.

El `&` obtiene su dirección:

```c
Course *
```

Entonces:

```c
evaluate_course_eligibility(
    &catalog->courses[i],
    history
);
```

permite modificar directamente ese curso almacenado dentro del catálogo.

---

## «¿Qué significa `course->code`?»

`course` es:

```c
Course *
```

Entonces:

```c
course->code
```

es equivalente a:

```c
(*course).code
```

`->` permite acceder a campos mediante un puntero.

---

## «¿Por qué algunos parámetros tienen `const`?»

Ejemplo:

```c
const StudentHistory *history
```

Significa que esa función recibe el historial para consultarlo y no debería modificarlo.

Lo mismo ocurre con:

```c
const Catalog *catalog
```

en el serializer.

---

## «Explíqueme el proyecto completo empezando desde `main()`.»

Una respuesta compacta sería:

> `main()` inicializa un `Catalog` y un `StudentHistory`. Luego `parse_catalog()` lee el TSV, valida cada columna y construye dinámicamente los `Course`, `Group` y `Schedule`. Después `parse_student_history()` lee los códigos aprobados y los almacena en un `CodeList`, comprobando que existan en el catálogo. El detector recorre los horarios de grupos de cursos diferentes y almacena los conflictos. Elegibilidad compara requisitos y correquisitos contra el historial. Finalmente el serializer recorre el catálogo ya procesado y escribe el JSON. Al terminar, `student_history_free()` y `catalog_free()` liberan toda la memoria dinámica.

Esa respuesta conecta prácticamente todo el código del proyecto.