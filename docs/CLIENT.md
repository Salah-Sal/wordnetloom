# Client Application Guide

This document provides detailed information about the WordNetLoom desktop client application.

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Getting Started](#getting-started)
- [User Interface](#user-interface)
- [Features](#features)
- [Package Structure](#package-structure)
- [Key Components](#key-components)
- [Event System](#event-system)
- [Customization](#customization)

## Overview

The WordNetLoom client is a JavaFX desktop application that provides a rich graphical interface for working with wordnet data. It uses the MVVM (Model-View-ViewModel) architectural pattern with the mvvmfx framework.

### Key Technologies

| Technology | Version | Purpose |
|------------|---------|---------|
| JavaFX | 11 | UI framework |
| mvvmfx | 1.8.0 | MVVM framework |
| JUNG | 2.0.1 | Graph visualization |
| RESTEasy | 3.6.1 | REST client |
| Weld | 3.0.0 | CDI implementation |
| ControlsFX | 11.0.2 | Enhanced UI controls |

## Architecture

### MVVM Pattern

```
┌────────────────────────────────────────────────────────────────┐
│                         VIEW LAYER                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │  FXML Files  │  │  CSS Styles  │  │  View Classes        │  │
│  │  (*.fxml)    │  │ wordnetloom  │  │  (*View.java)        │  │
│  │              │  │    .css      │  │                      │  │
│  └──────┬───────┘  └──────────────┘  └──────────┬───────────┘  │
│         │                                        │              │
│         │ Data Binding                          │ Code-behind  │
│         ▼                                        ▼              │
│  ┌────────────────────────────────────────────────────────────┐│
│  │                    VIEWMODEL LAYER                          ││
│  │  ┌──────────────────┐  ┌────────────────────────────────┐  ││
│  │  │  *ViewModel.java │  │  Properties & Commands         │  ││
│  │  │                  │  │  - StringProperty lemma        │  ││
│  │  │                  │  │  - Command searchCommand       │  ││
│  │  └────────┬─────────┘  └────────────────────────────────┘  ││
│  │           │                                                 ││
│  └───────────┼─────────────────────────────────────────────────┘│
│              │                                                   │
│              │ Service Calls                                     │
│              ▼                                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                    MODEL LAYER                              │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐ │ │
│  │  │ RemoteService│  │ Model Classes│  │ CDI Events       │ │ │
│  │  │              │  │ (Sense, etc.)│  │                  │ │ │
│  │  └──────────────┘  └──────────────┘  └──────────────────┘ │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### Component Interaction

```
┌─────────────┐      ┌─────────────────┐      ┌───────────────┐
│   User      │─────►│    View         │─────►│  ViewModel    │
│   Action    │      │   (FXML)        │      │               │
└─────────────┘      └─────────────────┘      └───────┬───────┘
                                                      │
                                                      ▼
                     ┌─────────────────┐      ┌───────────────┐
                     │    CDI Event    │◄─────│ RemoteService │
                     │    Bus          │      │   (REST)      │
                     └────────┬────────┘      └───────────────┘
                              │
                              ▼
                     ┌─────────────────┐
                     │  Other          │
                     │  ViewModels     │
                     └─────────────────┘
```

## Getting Started

### Prerequisites

- Java 11 JDK
- JavaFX 11 SDK
- Maven 3.x

### Building the Client

```bash
# Build from project root
mvn clean package -pl wordnetloom-client

# Output location
# wordnetloom-client/target/wordnetloom-client-3.0.jar
# wordnetloom-client/target/wordnetloom-client-3.0-dist.zip
```

### Running the Client

```bash
# With module path
java --module-path /path/to/javafx-sdk-11/lib \
     --add-modules javafx.controls,javafx.fxml,javafx.swing,javafx.web \
     -jar wordnetloom-client-3.0.jar

# Or extract the distribution
unzip wordnetloom-client-3.0-dist.zip
cd wordnetloom-client-3.0
java -jar wordnetloom-client-3.0.jar
```

### Configuration

Edit `application.properties` before running:

```properties
# Server connection
server.host=http://localhost:8080

# Lexicon icons (optional)
lexicon.1=polish.png
lexicon.2=english.png
lexicon.3=english.png
```

## User Interface

### Main Window Layout

```
┌─────────────────────────────────────────────────────────────────────┐
│  Menu Bar                                                           │
├─────────────────────────────────────────────────────────────────────┤
│  Toolbar                                                            │
├────────────────────┬────────────────────────────────────────────────┤
│                    │                                                │
│   Search Panel     │            Work Area                           │
│                    │                                                │
│  ┌──────────────┐  │  ┌────────────────────────────────────────┐   │
│  │ Search Form  │  │  │  Sense/Synset Details                  │   │
│  │              │  │  │                                        │   │
│  │ - Lemma      │  │  │  - Properties                          │   │
│  │ - Lexicon    │  │  │  - Relations                           │   │
│  │ - POS        │  │  │  - Examples                            │   │
│  │ - Domain     │  │  │                                        │   │
│  └──────────────┘  │  └────────────────────────────────────────┘   │
│                    │                                                │
│  ┌──────────────┐  │  ┌────────────────────────────────────────┐   │
│  │ Results List │  │  │  Graph Visualization                   │   │
│  │              │  │  │                                        │   │
│  │ - Sense 1    │  │  │       [Synset]                         │   │
│  │ - Sense 2    │  │  │          │                             │   │
│  │ - Sense 3    │  │  │    ┌─────┼─────┐                       │   │
│  │              │  │  │    ▼     ▼     ▼                       │   │
│  └──────────────┘  │  │ [Hyper] [Hypo] [Mero]                  │   │
│                    │  │                                        │   │
│                    │  └────────────────────────────────────────┘   │
│                    │                                                │
├────────────────────┴────────────────────────────────────────────────┤
│  Status Bar                                                         │
└─────────────────────────────────────────────────────────────────────┘
```

### Key UI Components

1. **Login Dialog**: Authentication screen
2. **Search Panel**: Search form and results
3. **Sense Form**: Edit sense properties
4. **Synset Form**: Edit synset properties
5. **Relation Form**: Manage relations
6. **Graph View**: Visualize semantic network
7. **Properties Panel**: View/edit attributes

## Features

### Search

- Search by lemma (word)
- Filter by lexicon
- Filter by part of speech
- Filter by domain
- Paginated results

### Sense Management

- Create new senses
- Edit sense properties
- Change sense position in synset
- Attach/detach from synsets
- Add usage examples

### Synset Management

- Create synsets
- Edit synset properties (definition, etc.)
- View member senses
- Add/remove senses

### Relation Management

- View sense relations
- View synset relations
- Create new relations
- Delete relations
- Filter by relation type

### Graph Visualization

- Interactive graph display
- Pan and zoom
- Node expansion
- Relation type coloring
- Path to hypernym

## Package Structure

```
pl.edu.pwr.wordnetloom.client/
│
├── Application.java                 # Entry point
│
├── config/
│   └── ResourceProvider.java        # CDI producers
│
├── events/                          # CDI event classes
│   ├── LoadGraphEvent.java
│   ├── OpenMainApplicationEvent.java
│   ├── RefreshSenseEvent.java
│   ├── ShowSenseRelationsEvent.java
│   ├── TriggerShutdownEvent.java
│   └── ... (15+ events)
│
├── model/                           # Data transfer objects
│   ├── Dictionary.java
│   ├── Lexicon.java
│   ├── Sense.java
│   ├── SenseRelation.java
│   ├── Synset.java
│   ├── SynsetRelation.java
│   ├── User.java
│   └── ... (30+ models)
│
├── service/                         # Backend services
│   ├── RemoteService.java           # REST client
│   ├── Dictionaries.java            # Dictionary cache
│   └── RelationTypeService.java     # Relation type cache
│
└── ui/                              # UI components
    ├── main/
    │   ├── MainView.java
    │   ├── MainView.fxml
    │   └── MainViewModel.java
    │
    ├── logindialog/
    │   ├── LoginDialogView.java
    │   ├── LoginDialogView.fxml
    │   └── LoginDialogViewModel.java
    │
    ├── search/
    │   ├── SearchView.java
    │   ├── SearchView.fxml
    │   └── SearchViewModel.java
    │
    ├── senseform/
    │   ├── SenseFormView.java
    │   ├── SenseFormView.fxml
    │   └── SenseFormViewModel.java
    │
    ├── synsetform/
    │   ├── SynsetFormView.java
    │   ├── SynsetFormView.fxml
    │   └── SynsetFormViewModel.java
    │
    ├── graph/
    │   ├── GraphView.java
    │   └── GraphViewModel.java
    │
    ├── relationform/
    │   └── ... relation UI
    │
    ├── alerts/
    │   └── ... alert dialogs
    │
    ├── dialogs/
    │   └── ... common dialogs
    │
    └── ... (30+ UI packages)
```

## Key Components

### Application Entry Point

`Application.java` - Main class extending `MvvmfxCdiApplication`:

```java
public class Application extends MvvmfxCdiApplication {

    public static void main(String... args) {
        Locale.setDefault(Locale.ENGLISH);
        launch(args);
    }

    @Override
    public void startMvvmfx(Stage stage) throws Exception {
        // Load login dialog
        ViewTuple<LoginDialogView, LoginDialogViewModel> load =
            FluentViewLoader.fxmlView(LoginDialogView.class).load();

        // Show dialog
        DialogHelper.showDialog(load.getView(), stage, "/wordnetloom.css");
    }

    public void triggerShutdown(@Observes TriggerShutdownEvent event) {
        Platform.exit();
    }
}
```

### RemoteService

Central service for all REST API communication (`service/RemoteService.java`):

```java
@Singleton
public class RemoteService {

    private Client client;

    @PostConstruct
    public void init() {
        client = ClientBuilder.newClient();
    }

    // Authentication
    public Optional<User> login(String email, String password) { ... }

    // Sense operations
    public List<Sense> findSenses(SearchFilter filter) { ... }
    public Optional<Sense> findSense(UUID id) { ... }
    public void saveSense(Sense sense) { ... }
    public void deleteSense(UUID id) { ... }

    // Synset operations
    public List<Synset> findSynsets(SearchFilter filter) { ... }
    public Optional<Synset> findSynset(UUID id) { ... }

    // Graph operations
    public NodeExpanded getSenseGraph(UUID id) { ... }
    public NodeExpanded getSynsetGraph(UUID id) { ... }

    // Dictionaries
    public List<Dictionary> getDictionaries(String type) { ... }
}
```

### ViewModel Example

ViewModels manage presentation state and logic:

```java
public class SearchViewModel implements ViewModel {

    // Observable properties for data binding
    private final StringProperty lemma = new SimpleStringProperty();
    private final ObjectProperty<Lexicon> selectedLexicon = new SimpleObjectProperty<>();
    private final ObservableList<SenseRow> results = FXCollections.observableArrayList();

    @Inject
    RemoteService remoteService;

    @Inject
    Event<LoadGraphEvent> loadGraphEvent;

    // Commands
    public void search() {
        SearchFilter filter = new SearchFilter();
        filter.setLemma(lemma.get());
        filter.setLexicon(selectedLexicon.get());

        List<Sense> senses = remoteService.findSenses(filter);
        results.setAll(senses.stream().map(SenseRow::new).collect(toList()));
    }

    public void selectSense(Sense sense) {
        loadGraphEvent.fire(new LoadGraphEvent(sense.getSynsetId()));
    }

    // Property accessors for binding
    public StringProperty lemmaProperty() { return lemma; }
    public ObservableList<SenseRow> getResults() { return results; }
}
```

### View Example

Views are defined in FXML with code-behind:

```xml
<!-- SearchView.fxml -->
<VBox xmlns:fx="http://javafx.com/fxml" fx:controller="...SearchView">
    <TextField fx:id="lemmaField" promptText="Enter word..."/>
    <ComboBox fx:id="lexiconCombo"/>
    <Button text="Search" onAction="#onSearch"/>

    <TableView fx:id="resultsTable">
        <columns>
            <TableColumn text="Lemma" fx:id="lemmaColumn"/>
            <TableColumn text="POS" fx:id="posColumn"/>
        </columns>
    </TableView>
</VBox>
```

```java
// SearchView.java
public class SearchView implements FxmlView<SearchViewModel> {

    @FXML private TextField lemmaField;
    @FXML private TableView<SenseRow> resultsTable;

    @InjectViewModel
    private SearchViewModel viewModel;

    public void initialize() {
        // Bind UI to ViewModel
        lemmaField.textProperty().bindBidirectional(viewModel.lemmaProperty());
        resultsTable.setItems(viewModel.getResults());
    }

    @FXML
    public void onSearch() {
        viewModel.search();
    }
}
```

## Event System

The client uses CDI events for loose coupling between components.

### Available Events

| Event | Purpose |
|-------|---------|
| `LoadGraphEvent` | Request graph visualization |
| `OpenMainApplicationEvent` | Open main window after login |
| `RefreshSenseEvent` | Refresh sense display |
| `ShowSenseRelationsEvent` | Show relations panel |
| `TriggerShutdownEvent` | Request application shutdown |
| `UpdateSenseEvent` | Sense was updated |
| `UpdateSynsetEvent` | Synset was updated |

### Firing Events

```java
@Inject
Event<LoadGraphEvent> loadGraphEvent;

public void onSenseSelected(Sense sense) {
    loadGraphEvent.fire(new LoadGraphEvent(sense.getSynsetId()));
}
```

### Observing Events

```java
public void onLoadGraph(@Observes LoadGraphEvent event) {
    UUID synsetId = event.getSynsetId();
    loadGraph(synsetId);
}
```

## Customization

### Styling

The application uses CSS for styling. Edit `wordnetloom.css`:

```css
/* Main theme colors */
.root {
    -fx-base: #f0f0f0;
    -fx-accent: #0078d4;
}

/* Custom button style */
.primary-button {
    -fx-background-color: #0078d4;
    -fx-text-fill: white;
}

/* Graph node styling */
.graph-node {
    -fx-background-radius: 5;
    -fx-padding: 5;
}
```

### Adding New Views

1. Create FXML file:
   ```xml
   <!-- MyView.fxml -->
   <VBox xmlns:fx="http://javafx.com/fxml" fx:controller="...MyView">
       <!-- UI content -->
   </VBox>
   ```

2. Create View class:
   ```java
   public class MyView implements FxmlView<MyViewModel> {
       @InjectViewModel
       private MyViewModel viewModel;
   }
   ```

3. Create ViewModel:
   ```java
   public class MyViewModel implements ViewModel {
       // State and logic
   }
   ```

4. Load the view:
   ```java
   ViewTuple<MyView, MyViewModel> tuple =
       FluentViewLoader.fxmlView(MyView.class).load();
   ```

### Adding Icons

Place icon files in `src/main/resources/icons/` and reference in `application.properties`:

```properties
lexicon.4=danish.png
```

## Troubleshooting

### Common Issues

**JavaFX not found:**
Ensure JavaFX SDK is in the module path.

**Connection refused:**
Check server URL in `application.properties`.

**Login fails:**
Verify credentials and server connectivity.

**Graph not displaying:**
Check browser compatibility for WebView components.

### Logging

The client uses SLF4J with Logback. Configure in `logback.xml`:

```xml
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss} %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="STDOUT"/>
    </root>

    <!-- Debug REST calls -->
    <logger name="pl.edu.pwr.wordnetloom.client.service" level="DEBUG"/>
</configuration>
```
