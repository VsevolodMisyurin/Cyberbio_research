**🧬 Cyberbio_research**

**Cyberbio_cell* is an experimental artificial‐cell model providing flexible simulation of gene expression, regulatory logic, metabolism, toxin dynamics, and cell division. It’s well suited for research, regulatory-network modeling, and teaching the fundamentals of systems biology.

> The project allows you to create virtual genes, define their behavior, activation/deactivation conditions, protein effects, and toxin influence—with support for visualization and analysis.

**📦 Features**

- ✅ Gene expression and protein synthesis modeling, including:
  - Activation and deactivation conditions
  - Promoters and enhancers (influence expression rate and limit)
  - Delayed protein effects (take effect on the next tick)
  - Protein accumulation and stability

- ✅ Protein effects implementation, such as:
  - Energy generation
  - Conversion of other proteins into energy
  - Conversion of TPM into energy
  - Toxin detection
  - Toxin degradation
  - Mutation detection
  - Mutation repair
  - Generation of cell products based on the responsible protein amount
  - Triggering cell division

- ✅ Cell death based on energy level via a logistic function:
  ```python
  def cell_death_probability(energy: float) -> float:
      x, k, x0 = abs(energy), 3.0, 3.5
      return 1.0 / (1.0 + np.exp(-k * (x - x0)))
  ```
  - Before each division step, every cell has a probability p_death of dying (based on sampling or binomial approximation).

- ✅ Toxin effects, including:
  - Energy suppression
  - Reduced TPM expression
  - Parameterizable effect strength via a sigmoid function

- ✅ Mutagenesis and repair:
  - Random point mutations (threshold, promoter, enhancer) with a defined mutation_chance
  - Mutation sensor (mutsens) detects deviations from the reference genome
  - Repair enzyme (mutrep) attempts to restore or re-mutate genes

- ✅ Flexible architecture: freely define any number of genes and configure interaction logic.

- ✅ Built-in DataFrame log with complete history of gene expression and toxin levels.

- ✅ New log types: mut_detected, tox_detected + final genome state after simulation.

  ---

**🧪 Example simulation run**

```python
df, all_genes = run_simulation(
    ticks=3000,             # desired number of simulation ticks (maximum 3000 steps in this example)
    initial_food=1.0,       # initial level of "food" in the environment
    initial_energy=0.0,     # initial amount of energy inside the cell
    initial_cell_num=10,    # starting number of cells in the population
    support_cell_num=1000,  # "capacity" of the environment: maximum number of cells with a comfortable supply of "food"
    energy_cost=0.002,      # cost of each unit of transcript (TPM) in energy
    mutation_chance=0.0001, # probability of random mutation of each gene per tick
    protein_stability=0.5,  # base coefficient of stabilization of all proteins (0.0…1.0)
    cellular_products={
    },
    toxins={
        "Energotoxin": (-2, ("energy", 0.0))
    },
    receptors={
        "ENGSENS":   (12, "qual", "st", "av", 50, ["DIVISION == 'inactive'"], ["DIVISION == 'active'"], False),
        "FOODSENS":  (4,  "qual", "av", "av", 50, ["Energy <= 1", "KINFEED == 'active'"], ["Energy >= 2", "DIVISION == 'active'"], False),
        "MUTSENS":    (2, "qual", "st", "av", 50, ["ENGSENS == 'active'"], ["DIVISION == 'active'"], ("mutsens", 1)),
    },
    metabolism={
        "HARVEST":   (2,  "quan", "av", "av", 150, ["Energy <= -0.25", "FOODSENS == 'active'"], ["Energy >= 0.25", "Energy <= -2.9", "DIVISION == 'active'"], ("energy", 0.05)),
        "FEED":   (2,  "quan", "av", "av", 50, ["Energy <= -2"], ["Energy >= 2", "DIVISION == 'active'"], ("energy", 0.1)),
    },
    kinases={
        "KINFEED":     (3, "qual", "av", "wk", 50, ["Energy <= 1", "ENGSENS == 'active'"], ["Energy > 0.5", "DIVISION == 'active'"], False),
    },
    cell_cycle={
        "CDK1":      (3,  "qual", "av", "wk", 50, ["Energy >= 1"], ["DIVISION == 'active'"], False),
    },
    division={
        "MUTGUARD":  (2, "qual", "st", "av", 50, ["DIVISION == 'inactive'"], ["DIVISION == 'active'"], ("mutrep", 0.01)),
        "DIVISION":  (12, "qual", "av", "av", 50, ["CDK1 == 'active'"], ["DIVISION == 'active'"], ("div", 1)),
    }
)
df.head(15)
```

**📊 Visualization**
```python
plot_gene_dynamics(df.iloc[:200, :], title="Simple model with low mutagenesis, start of simulation",
                   genes={"FOODSENS": "blue", 
                          "FEED": "red", 
                          "HARVEST": "green",
                          "CDK1": "pink",
                          "DIVISION": "purple"})
```

**🔍 Gene format**
Each gene is defined as a tuple:

```python
"GENE": (
    threshold,         # activation threshold in pseudo-TPM units
    "qual"/"quan",     # type: qualitative (by threshold) or quantitative (from onset of expression)
    "wk"/"av"/"st",    # promoter strength: weak, average, strong (expression rate)
    "wk"/"av"/"st",    # enhancer strength: growth limit
    val,               # protein mass in kDa (affects stability)
    [on_conditions],   # activation conditions (all must be met at least once)
    [off_conditions],  # deactivation conditions (any one is enough)
    ("function", val)  # protein function and its strength, or False
)
```

Example protein functions:

("energy", 0.01) — produces energy based on TPM

("process", 0.1) — processes other proteins to yield energy

("RNAdigest", 0.05) — converts TPM into energy

("mutsens", p) — mutation sensor: scans all genes for deviation from reference with chance p, sets mut_detected = True if mismatch is found

("mutrep", p) — mutation repairer: attempts to restore genes or undergo mutation itself with chance p

("div", 1) — triggers cell division and spends a defined amount of energy

**⚗️ Toxins**
```python
toxins = {
    "Name": (σ, ("type", base_effect)),
}
```

"Name" — toxin name

σ — toxin level in the environment, ranging from -3 to 3

"type" — effect type: "energy" (suppresses energy), "TPM" (suppresses gene expression)

base_effect — effect strength (0 to 1)

Effect is calculated as:

effect = (2 * base_effect) / (1 - exp(-k * σ)

Example:
```python
toxins = {
    "Energotoxin": (-1, ("energy", 0.3)),
    "RNAtoxin": (2, ("TPM", 0.2))
}
```

**⚠️ Simulation stop conditions**

- All cells die (e.g., energy persistently outside [-3.0, 3.0] range)

- Reaching the set number of simulation ticks

**🛠️ TODO / Plans**
- Split code into classes: Gene, Protein, Cell
- Support additional gene classes
- GUI or interactive interface
- Mutation and evolution mechanisms
- Population-level modeling and intercellular signaling
- Automatic logging and serialization
- Pseudotranscriptome export for standard bioinformatics tools
- Expanding the model toward realistic cell processes

**📜 License**
MIT — free to use and extend, preferably with credit to the author.

**👨‍🔬 Author**

Vsevolod Misyurin — @VsevolodMisyurin

Cyberbiologist & Researcher




**Cyberbio_cell* — это экспериментальная модель искусственной клетки с гибкой симуляцией экспрессии генов, регуляции, метаболизма, токсинов и деления. Подходит для исследований, моделирования регуляторных сетей и обучения основам системной биологии.

> Проект позволяет создавать виртуальные гены, описывать их поведение, условия активации/деактивации, эффекты белков и влияние токсинов — с возможностью визуализации и анализа результата.

---

**📦 Возможности**

- ✅ Моделирование транскрипции и экспрессии генов, а также синтеза белков с учётом:
  - условий активации и деактивации;
  - промоторов и энхансеров (влияют на скорость и предел экспрессии);
  - задержки действия белков (эффект реализуется на следующем такте).
  - накопления белков и их стабильности

- ✅ Реализация эффектов белков:
  - генерация энергии;
  - переработка белков других генов в энергию;
  - переработка TPM в энергию;
  - детекция токсинов;
  - переработка токсинов;
  - детекция мутаций;
  - репарация мутаций;
  - генерация клеточных продуктов в зависимости от количества белка, ответственного за синтез продукта;
  - запуск деления клетки.

- ✅ **Гибель клеток**:
  - рассчитывается на основе уровня `energy` через логистическую функцию  
  ```python
  def cell_death_probability(energy: float) -> float:
      x, k, x0 = abs(energy), 3.0, 3.5
      return 1.0 / (1.0 + np.exp(-k * (x - x0)))
  ```
  - применяется перед делением: по каждому существующему «экземпляру» клетки с вероятностью p_death умирает, остальные выживают (сэмплирование или биномиальное приближение).

- ✅ Влияние токсинов:
  - подавление энергии;
  - снижение экспрессии TPM;
  - параметризуемая сила действия через сигмоидальную функцию.
    
- ✅ Мутагенез и репарация:
  - случайные точечные мутации порога, промотора или энхансера (mutation_chance);
  - сенсор мутаций (mutsens) фиксирует отклонения от эталонного генома;
  - репаратор (mutrep) восстанавливает или вновь мутирует гены.

- ✅ Гибкая архитектура: можно добавлять любое количество генов и настраивать логику их взаимодействия.

- ✅ Встроенный DataFrame лог с полной историей экспрессий и уровней токсинов.

- ✅ Логи нового типа: `mut_detected`, `tox_detected` + финальный набор генов после симуляции.


---

**🧪 Пример запуска**

```python
df, all_genes = run_simulation(
    ticks=3000,             # количество тактов симуляции (максимум 3000 шагов)
    initial_food=1.0,       # начальный уровень «еды» в окружении
    initial_energy=0.0,     # начальное количество энергии внутри клетки
    initial_cell_num=10,    # стартовое число клеток в популяции
    support_cell_num=1000,  # «ёмкость» среды: максимальное число клеток при комфортном обеспечении "едой"
    energy_cost=0.002,      # стоимость каждой единицы транскрипта (TPM) в энергии
    mutation_chance=0.0001, # вероятность случайной мутации каждого гена за такт
    protein_stability=0.5,  # базовый коэффициент стабилизации всех белков (0.0…1.0)
    cellular_products={
    },
    toxins={
        "Energotoxin": (-2, ("energy", 0.0))
    },
    receptors={
        "ENGSENS":   (12, "qual", "st", "av", 50, ["DIVISION == 'inactive'"], ["DIVISION == 'active'"], False),
        "FOODSENS":  (4,  "qual", "av", "av", 50, ["Energy <= 1", "KINFEED == 'active'"], ["Energy >= 2", "DIVISION == 'active'"], False),
        "MUTSENS":    (2, "qual", "st", "av", 50, ["ENGSENS == 'active'"], ["DIVISION == 'active'"], ("mutsens", 1)),
    },
    metabolism={
        "HARVEST":   (2,  "quan", "av", "av", 150, ["Energy <= -0.25", "FOODSENS == 'active'"], ["Energy >= 0.25", "Energy <= -2.9", "DIVISION == 'active'"], ("energy", 0.05)),
        "FEED":   (2,  "quan", "av", "av", 50, ["Energy <= -2"], ["Energy >= 2", "DIVISION == 'active'"], ("energy", 0.1)),
    },
    kinases={
        "KINFEED":     (3, "qual", "av", "wk", 50, ["Energy <= 1", "ENGSENS == 'active'"], ["Energy > 0.5", "DIVISION == 'active'"], False),
    },
    cell_cycle={
        "CDK1":      (3,  "qual", "av", "wk", 50, ["Energy >= 1"], ["DIVISION == 'active'"], False),
    },
    division={
        "MUTGUARD":  (2, "qual", "st", "av", 50, ["DIVISION == 'inactive'"], ["DIVISION == 'active'"], ("mutrep", 0.01)),
        "DIVISION":  (12, "qual", "av", "av", 50, ["CDK1 == 'active'"], ["DIVISION == 'active'"], ("div", 1)),
    }
)
df.head(15)
```

**📊 Визуализация**
```python
plot_gene_dynamics(df.iloc[:200, :], title="Simple model with low mutagenesis, start of simulation",
                   genes={"FOODSENS": "blue", 
                          "FEED": "red", 
                          "HARVEST": "green",
                          "CDK1": "pink",
                          "DIVISION": "purple"})
```

**🔍 Формат описания гена**
Каждый ген задаётся кортежем:

```python
"GENE": (
    threshold,         # порог активации в единицах псевдо-TPM
    "qual"/"quan",     # тип: качественный (по порогу) или количественный (от начала экспрессии)
    "wk"/"av"/"st",    # промотор: слабый, средний, сильный (скорость роста в единицах псевдо-TPM)
    "wk"/"av"/"st",    # энхансер: ограничение роста
    val,               # масса белка в kDa, влияющая на стабильность белка (белки массой 10 kDa "разваливаются" в четыре раза быстрее, чем белки с массой 10 кДа")
    [on_conditions],   # условия активации (все должны соблюдаться, но достаточно однократного соблюдения)
    [off_conditions],  # условия деактивации (достаточно одного)
    ("function", val)  # функция белка и её сила или False
)
```

Примеры функций белка:

("energy", 0.01) — добывает энергию в количестве единиц за каждую единицу псевдо-TPM гена

("process", 0.1) — перерабатывает белки других генов, и возвращает заданное количество единиц энергии за каждый псевдо-TPM переработанного белка

("RNAdigest", 0.05) — конвертирует TPM в энергию, и возвращает заданное количество единиц энергии за каждый псевдо-TPM переработанного белка

("mutsens", p) — мутационный сенсор: даёт до 3 + floor(TPM/12) попыток «просканировать» все гены, и с шансом p на попытку сравнить их с эталонным геномом; если найдена хоть одна отличающаяся мутация, поднимает `mut_detected = True`.

("mutrep", p) — репаратор: при поднятом `mut_detected` даёт такие же попытки (3 + floor(TPM/12)); для каждой отличающейся пары «текущий vs эталон» работает с шансом p:
    - восстанавливает эталонную последовательность, или
    - сама подвергается случайной мутации (метод `mutate_gene`).

("div", 1) — инициирует деление клетки и тратит заданное количество энергии

**⚗️ Токсины**
```python
toxins = {
    "Name": (σ, ("type", base_effect)),
}
```

"Name" - название токсина

σ — уровень токсина в среде, в диапазне от -3 до 3

"type" - действие токсина, поддерживаются типы "energy" — снижает уровень энергии, "TPM" — ограничивает рост экспрессии

base_effect - величина эффекта в диапазоне от 0 до 1

Эффект рассчитывается как:

effect = (2 * base_effect) / (1 - exp(-k * σ)

Пример
```python
toxins = {
    "Energotoxin": (-1, ("energy", 0.3)),
    "RNAtoxin": (2, ("TPM", 0.2))
}
```

**⚠️ Условия остановки симуляции**
- Гибель всех клеток популяции, если энергия надолго выходит за диапазон ниже -3.0 или выше 3.0.
- ​Прошло заданное количество тактов

**🛠️ TODO / планы**

- Разделение кода на классы: Gene, Protein, Cell
- Пооддерка дополнительных классов генов
- GUI или интерактивный интерфейс
- Механизм мутаций и эволюции
- Поддержка популяций и межклеточных сигналов
- Автоматическое логирование и сериализация симуляций
- Выделение псевдотранскриптомов для изучения в стандартных биологических пакетах
- Расширение модели, добавление новых функций, приближение к реальным клеточным процессам

**📜 Лицензия**
MIT — свободно используйте и развивайте, желательно с указанием авторства.

**👨‍🔬 Автор**

Всеволод Мисюрин — @VsevolodMisyurin

Исследователь-кибербиолог.
