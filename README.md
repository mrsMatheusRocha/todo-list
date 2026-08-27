# 📝 Todo List - Aplicativo de Gerenciamento de Tarefas

Um aplicativo Android moderno e elegante para gerenciamento de tarefas pessoais, desenvolvido com as mais recentes tecnologias do ecossistema Kotlin/Android. O app oferece uma experiência de usuário fluida com criação, edição, conclusão e exclusão de tarefas.

---

## 🎯 Objetivo da Aplicação

O **Todo List** foi desenvolvido para demonstrar as melhores práticas de desenvolvimento Android, incluindo:
- Arquitetura MVVM (Model-View-ViewModel)
- Gerenciamento reativo de estado com Flow e StateFlow
- Persistência de dados com Room
- Interface moderna e responsiva com Jetpack Compose

A aplicação permite que usuários mantenham o controle de suas tarefas de forma simples e intuitiva, com funcionalidades completas de CRUD (Create, Read, Update, Delete).

---

## 🛠 Tecnologias Utilizadas

### Core
- **Kotlin** - Linguagem de programação moderna e segura
- **Android 11+** (API 30+)

### UI & Navigation
- **Jetpack Compose** - Framework de UI declarativo para construir interfaces modernas
- **Navigation Compose** - Gerenciamento de navegação entre telas

### Data & State Management
- **Room** - Banco de dados local SQLite com abstração de alto nível
- **Coroutines** - Programação assíncrona simplificada
- **Flow/StateFlow** - Streams reativos para observar mudanças de estado
- **ViewModel** - Gerenciamento de estado da UI lifecycle-aware

---

## 📐 Arquitetura e Componentes

### Estrutura de Camadas

```
┌─────────────────────────────────┐
│     UI Layer (Composables)      │
│  ListaTarefasScreen             │
│  FormularioTarefaScreen         │
├─────────────────────────────────┤
│    ViewModel Layer              │
│  TarefaViewModel                │
├─────────────────────────────────┤
│    Repository Layer             │
│  TarefaRepository               │
├─────────────────────────────────┤
│    Data Layer                   │
│  TarefaDatabase + TarefaDao     │
└─────────────────────────────────┘
```

---

## 🔄 Explicação dos Componentes Principais

### 📦 TarefaRepository

**Responsabilidade**: Atuar como intermediário entre a ViewModel e a camada de dados.

```kotlin
class TarefaRepository(private val dao: TarefaDao) {
    val tarefas: Flow<List<Tarefa>> = dao.listarTodas()
    
    suspend fun inserir(tarefa: Tarefa) = dao.inserir(tarefa)
    suspend fun atualizar(tarefa: Tarefa) = dao.atualizar(tarefa)
    suspend fun deletar(tarefa: Tarefa) = dao.deletar(tarefa)
}
```

**Funcionalidades**:
- **Abstração do DAO**: Encapsula toda a lógica de acesso ao banco de dados
- **Flow de Tarefas**: Expõe um `Flow<List<Tarefa>>` que emite atualizações automáticas
- **Operações Suspensas**: Métodos `suspend` para inserção, atualização e deleção que rodam em threads de I/O

**Vantagens**:
- Desacopla a ViewModel do Room
- Facilita testes com mock do repository
- Centraliza a lógica de persistência

---

### 🎬 TarefaViewModel

**Responsabilidade**: Gerenciar o estado da UI e coordenar ações entre a View e o Repository.

```kotlin
class TarefaViewModel(private val repository: TarefaRepository) : ViewModel() {
    
    val tarefas: StateFlow<List<Tarefa>> = repository.tarefas
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            initialValue = emptyList()
        )

    fun inserir(tarefa: Tarefa) = viewModelScope.launch { 
        repository.inserir(tarefa) 
    }
    fun atualizar(tarefa: Tarefa) = viewModelScope.launch { 
        repository.atualizar(tarefa) 
    }
    fun deletar(tarefa: Tarefa) = viewModelScope.launch { 
        repository.deletar(tarefa) 
    }
}
```

**Funcionalidades**:
- **StateFlow de Tarefas**: Converte o `Flow` do repository em `StateFlow` para uso na Compose
- **Lifecycle-Aware**: Usa `viewModelScope` para cancelar coroutines automaticamente
- **Operações Assíncronas**: Dispara operações do repository em background threads

**Ciclo de Vida**:
1. O `StateFlow` mantém o valor mais recente de tarefas
2. `SharingStarted.WhileSubscribed(5_000)` garante que a coroutine continua por 5 segundos após não haver subscribers
3. A ViewModel é destruída com a Activity/Fragment, cancelando todas as coroutines

---

### 📋 ListaTarefasScreen

**Responsabilidade**: Observar o estado das tarefas e disparar ações de edição, deleção e criação.

#### Como observa o estado:

```kotlin
@Composable
fun ListaTarefasScreen(
    viewModel: TarefaViewModel,
    onNovaTarefa: () -> Unit,
    onEditarTarefa: (Int) -> Unit
) {
    val tarefas by viewModel.tarefas.collectAsStateWithLifecycle()
    // ...
}
```

- **`collectAsStateWithLifecycle()`**: Hook especial do Compose que coleta o `StateFlow` de forma lifecycle-aware
- Automaticamente para de coletar quando a tela é destruída

#### Como dispara ações:

```kotlin
ListaTarefasContent(
    tarefas = tarefas,
    onNovaTarefa = onNovaTarefa,           // Navegar para formulário
    onEditarTarefa = onEditarTarefa,       // Navegar para edição
    onCheckedChange = { tarefa, concluida ->
        viewModel.atualizar(tarefa.copy(concluida = concluida))
    },                                     // Marcar como concluída/pendente
    onDeletar = { tarefa -> viewModel.deletar(tarefa) }  // Deletar
)
```

#### Componentes UI:

- **TopAppBar**: Exibe título "Minhas Tarefas"
- **FloatingActionButton**: Botão flutuante para criar nova tarefa
- **LazyColumn**: Lista vertical otimizada com scroll
- **TarefaItem**: Card individual mostrando:
  - Checkbox para marcar conclusão
  - Título com risco se concluída
  - Descrição (máximo 1 linha)
  - Ícone de deleção

---

### ✏️ FormularioTarefaScreen

**Responsabilidade**: Diferençiar entre modo de criação (nova tarefa) e modo de edição (tarefa existente).

#### Como diferencia cadastro e edição:

```kotlin
@Composable
fun FormularioTarefaScreen(
    viewModel: TarefaViewModel,
    tarefaId: Int,
    onVoltar: () -> Unit
) {
    val tarefas by viewModel.tarefas.collectAsStateWithLifecycle()
    
    // Se tarefaId == 0: nova tarefa
    // Se tarefaId != 0: editar tarefa existente
    val tarefaExistente = remember(tarefas, tarefaId) {
        tarefas.find { it.id == tarefaId }
    }

    FormularioTarefaContent(
        isEdicao = tarefaId != 0,  // Flag de edição
        tituloInicial = tarefaExistente?.titulo ?: "",
        descricaoInicial = tarefaExistente?.descricao ?: "",
        onSalvar = { titulo, descricao ->
            if (tarefaId == 0) {
                // CRIAR: Nova tarefa sem ID
                viewModel.inserir(Tarefa(titulo = titulo, descricao = descricao))
            } else {
                // EDITAR: Atualizar tarefa existente
                tarefaExistente?.let {
                    viewModel.atualizar(it.copy(titulo = titulo, descricao = descricao))
                }
            }
            onVoltar()
        },
        onVoltar = onVoltar
    )
}
```

#### Diferenças na UI:

```kotlin
TopAppBar(
    title = { 
        Text(if (isEdicao) "Editar Tarefa" else "Nova Tarefa") 
    }
)
```

| Aspecto | Nova Tarefa | Editar Tarefa |
|---------|------------|---------------|
| Título da TopBar | "Nova Tarefa" | "Editar Tarefa" |
| Campos | Vazios | Preenchidos com dados existentes |
| Comportamento ao Salvar | Insere novo registro | Atualiza registro existente |
| ID | Gerado automaticamente | Mantém ID original |

---

### 🗺️ AppNavigation

**Responsabilidade**: Configurar as rotas e gerenciar a navegação entre telas.

```kotlin
@Composable
fun AppNavigation(viewModel: TarefaViewModel) {
    val navController = rememberNavController()

    NavHost(navController = navController, startDestination = "lista") {
        // Rota 1: Lista de Tarefas (tela inicial)
        composable("lista") {
            ListaTarefasScreen(
                viewModel = viewModel,
                onNovaTarefa = { navController.navigate("formulario/0") },
                onEditarTarefa = { id -> navController.navigate("formulario/$id") }
            )
        }
        
        // Rota 2: Formulário (com ID da tarefa como argumento)
        composable("formulario/{tarefaId}") { backStackEntry ->
            val tarefaId = backStackEntry.arguments?.getString("tarefaId")?.toInt() ?: 0
            FormularioTarefaScreen(
                viewModel = viewModel,
                tarefaId = tarefaId,
                onVoltar = { navController.popBackStack() }
            )
        }
    }
}
```

#### Rotas Configuradas:

| Rota | Descrição | Argumentos |
|------|-----------|-----------|
| `lista` | Tela inicial com lista de tarefas | - |
| `formulario/{tarefaId}` | Tela de formulário | `tarefaId` (String) |

#### Fluxo de Navegação:

```
┌──────────────────┐
│  Lista de Tarefas│
├──────────────────┤
│ Nova Tarefa      │ ──→ navigate("formulario/0")
│ Editar Tarefa    │ ──→ navigate("formulario/{id}")
└──────────────────┘
         ↑
         │
┌──────────────────┐
│  Formulário      │
├──────────────────┤
│ Voltar           │ ──→ popBackStack()
└──────────────────┘
```

#### Passagem de ID da Tarefa:

- **Para nova tarefa**: `formulario/0` (ID = 0 indica criação)
- **Para edição**: `formulario/{id}` (substitui `{id}` pelo ID real)
- **Recuperação**: `backStackEntry.arguments?.getString("tarefaId")?.toInt() ?: 0`

---

### 🚀 MainActivity

**Responsabilidade**: Inicializar a ViewModel e fornecer a navegação.

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        setContent {
            TodolistTheme {
                // Criar ViewModel usando factory
                val viewModel: TarefaViewModel = viewModel(
                    factory = TarefaViewModel.factory(applicationContext)
                )
                // Iniciar navegação
                AppNavigation(viewModel = viewModel)
            }
        }
    }
}
```

#### Processo de Inicialização:

1. **`enableEdgeToEdge()`**: Permite que a UI se estenda até as bordas da tela
2. **`setContent {}`**: Define o conteúdo Compose da Activity
3. **`viewModel(factory = ...)`**: Cria ViewModel com factory customizado que:
   - Obtém contexto da Activity
   - Acessa/cria o banco de dados
   - Instancia o DAO
   - Cria o Repository
   - Retorna a ViewModel pronta
4. **`AppNavigation(viewModel)`**: Passa a ViewModel para o sistema de navegação

#### Factory Pattern:

```kotlin
companion object {
    fun factory(context: Context): ViewModelProvider.Factory =
        object : ViewModelProvider.Factory {
            @Suppress("UNCHECKED_CAST")
            override fun <T : ViewModel> create(modelClass: Class<T>): T {
                val dao = TarefaDatabase.getDatabase(context).tarefaDao()
                return TarefaViewModel(TarefaRepository(dao)) as T
            }
        }
}
```

---

## 🚀 Como Executar o Projeto

### Pré-requisitos

- **Android Studio** (versão 2024.1 ou superior)
- **Java Development Kit (JDK)** 11 ou superior
- **Android SDK** com minSdk 24 (Android 7.0) ou superior
- **Emulador Android** ou dispositivo físico

### Passos para Instalação

1. **Clone o repositório**
   ```bash
   git clone https://github.com/mrsMatheusRocha/todo-list.git
   cd todo-list
   ```

2. **Abra no Android Studio**
   - File → Open → Selecione a pasta `todo-list`
   - Aguarde o Gradle sincronizar as dependências

3. **Sincronize as Dependências**
   ```bash
   ./gradlew clean build
   ```

4. **Execute no Emulador ou Dispositivo**
   - Emulador: `./gradlew installDebug && ./gradlew runDebug`
   - Ou selecione "Run 'app'" no Android Studio (Shift + F10)

5. **Explore a Aplicação**
   - Tela inicial exibe lista de tarefas
   - Clique em "+" para criar nova tarefa
   - Clique em uma tarefa para editar
   - Marque o checkbox para concluir
   - Clique no ícone de lixo para deletar

---

## 📱 Fluxo de Uso da Aplicação

### Criar Nova Tarefa
1. Clique no botão flutuante "+" 
2. Preencha o título (obrigatório) e descrição
3. Clique em "Salvar"
4. Nova tarefa aparece no topo da lista

### Editar Tarefa
1. Clique na tarefa que deseja editar
2. Modifique os campos
3. Clique em "Salvar"
4. Alterações são refletidas imediatamente

### Marcar como Concluída
1. Na lista, marque o checkbox ao lado da tarefa
2. O título aparecerá com risco
3. Desmarque para pendente novamente

### Deletar Tarefa
1. Clique no ícone de lixo na tarefa
2. Tarefa é removida imediatamente
3. Não há confirmação (considere implementar em melhorias)

---

## 📸 Evidências e Screenshots

### Tela de Lista de Tarefas
<img alt="Tela Inicial.png" height="400" src="docs/Tela%20Inicial.png"/>

### Tela de Formulário - Nova Tarefa
<img alt="Criação de Tarefa.png" height="400" src="docs/Cria%C3%A7%C3%A3o%20de%20Tarefa.png"/>

### Tela de Formulário - Editar Tarefa
<img alt="Edição de Tarefa.png" height="400" src="docs/Edi%C3%A7%C3%A3o%20de%20Tarefa.png"/>

### Tela de Formulário - Conclusão Tarefa
<img alt="Tarefa Concluida.png" height="400" src="docs/Tarefa%20Concluida.png"/>

### Tela de Build - Sem Erros
<img alt="Build sem Erros.png" height="400" src="docs/Build%20sem%20Erros.png"/>

---

## 🏗️ Estrutura de Arquivos

```
todo-list/
├── app/
│   ├── src/
│   │   ├── main/
│   │   │   ├── java/com/github/mrsmatheusrocha/todo_list/
│   │   │   │   ├── MainActivity.kt              # Activity principal
│   │   │   │   ├── data/
│   │   │   │   │   ├── Tarefa.kt               # Entidade Room
│   │   │   │   │   ├── TarefaDatabase.kt       # Banco de dados
│   │   │   │   │   └── TarefaDAO.kt            # Data Access Object
│   │   │   │   ├── repository/
│   │   │   │   │   └── TarefaRepository.kt     # Camada de repositório
│   │   │   │   ├── viewmodel/
│   │   │   │   │   └── TarefaViewModel.kt      # ViewModel
│   │   │   │   ├── navigation/
│   │   │   │   │   └── AppNavigation.kt        # Configuração de rotas
│   │   │   │   └── ui/
│   │   │   │       ├── ListaTarefasScreen.kt   # Tela de lista
│   │   │   │       └── FormularioTarefaScreen.kt # Tela de formulário
│   │   │   └── AndroidManifest.xml
│   │   └── androidTest/
│   │       └── java/.../data/
│   │           └── TarefaDAOTeste.kt          # Testes do DAO
│   └── build.gradle.kts
├── gradle/
├── build.gradle.kts
├── settings.gradle.kts
└── README.md
```

---

## 🔍 Modelo de Dados

### Entidade Tarefa

```kotlin
@Entity(tableName = "tarefas")
data class Tarefa(
    @PrimaryKey(autoGenerate = true) val id: Int = 0,
    val titulo: String,
    val descricao: String,
    val concluida: Boolean = false,
    val dataCriacao: Long = System.currentTimeMillis()
)
```

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `id` | Int | Chave primária, gerada automaticamente |
| `titulo` | String | Título obrigatório da tarefa |
| `descricao` | String | Descrição adicional |
| `concluida` | Boolean | Status de conclusão (padrão: false) |
| `dataCriacao` | Long | Timestamp de criação |

---

## 🧪 Testes

O projeto inclui testes instrumentados para o DAO:

```bash
./gradlew connectedAndroidTest
```

**Testes Inclusos**:
- ✅ Inserir e listar tarefas
- ✅ Marcar tarefa como concluída
- ✅ Deletar tarefa
- ✅ Banco de dados em memória para testes rápidos

---

## 🎨 Tema e Design

- **Material Design 3**: Componentes modernos do Material
- **Cores**: Tema claro com paleta do Material Design
- **Tipografia**: Tipos de letra standardizados
- **Responsividade**: Layout adaptável para diferentes tamanhos de tela

---

## 📚 Dependências Principais

```kotlin
// Jetpack & Compose
implementation(libs.androidx.activity.compose)
implementation(libs.androidx.lifecycle.runtime.ktx)
implementation(libs.androidx.compose.material3)
implementation(libs.androidx.navigation.compose)

// Room
implementation(libs.androidx.room.runtime)
implementation(libs.androidx.room.ktx)
ksp(libs.androidx.room.compiler)

// Coroutines
implementation(libs.androidx.lifecycle.runtime.ktx)
```

---

## 👤 Autor

**Desenvolvedor**: Matheus Rocha  
**GitHub**: [@mrsMatheusRocha](https://github.com/mrsMatheusRocha)  
**Data**: 2026

---

**Desenvolvido com ❤️ usando Kotlin e Jetpack Compose**
