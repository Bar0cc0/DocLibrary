# OOP Design Patterns Cheat Sheet


## 1. Strategy Pattern

**Purpose**: Define a family of algorithms, encapsulate each one, and make them interchangeable.

**Implementation Example**:
```python
## a. (Strategy) Interface
class IStrategy(ABC):
	@abstractmethod
	def execute(self, data: Any) -> Any: pass

## b. Concrete implementations (strategies)
class ConcreteStrategyA(IStrategy):
	def execute(self, data: Any) -> None: 
		print(f"Executing strategy A with data: ", data)

class ConcreteStrategyB(IStrategy):
	def execute(self, data: Any) -> None: 
		print(f"Executing strategy B with data: ", data)

## c. Context class which depends on the strategy interface
class Context:
	def __init__(self, strategy: IStrategy) -> None:
		self.strategy = strategy
	def run(self, data: Any) -> None:
		self.strategy.execute(data)

## d. Main function
def main():
	data = "Some data"
	strategyA:IStrategy = ConcreteStrategyA()
	strategyB:IStrategy = ConcreteStrategyB()

	# Injecting dependencies at runtime
	context = Context(strategyA)
	context.run(data)
	context = Context(strategyB)
	context.run(data)
```

**Benefits**:
- Encapsulates different algorithms (validation methods)
- Algorithms are interchangeable
- Client code remains unchanged when adding new strategies

## 2. Factory Pattern

**Purpose**: Create objects without exposing the instantiation logic to the client.

**Implementation Example**:
```python
# a. Factory class manages the creation of pointers to concrete classes
class Factory(ABC):    
    def __init__(self):
        self._strategies = {}
        self._initialize_strategies()
    
    def _initialize_strategies(self) -> None:
        """Initialize the registry with default strategies."""
        self.register('strategy_A', ConcreteStrategyA())
        self.register('strategy_B', ConcreteStrategyB())
    
    def register(self, name: str, strategy: IStrategy) -> None:
        """Register a new strategy dynamically."""
        if name in self._strategies:
            raise ValueError(f"Strategy {name} already registered")
        self._strategies[name] = strategy
    
    def get_strategy(self, name: str) -> IStrategy:
        """Get a strategy by name."""
        if name not in self._strategies:
            raise ValueError(f"Unknown strategy: {name}")
        return self._strategies[name]
```

**Benefits**:
- Centralizes object creation logic
- Provides registry capability for strategies
- Supports runtime configuration of concrete implementations

## 3. Dependency Injection

**Purpose**: Provide the dependencies of a class from the outside rather than having the class create them.

**Implementation Example**:
```python
class StrategyC(IStrategy):
    def __init__(self, config_provider: IConfigProvider):
        self._config = config_provider
        self._logger = config_provider.get_logger()
        # ...other initialization
```

**Benefits**:
- Reduces coupling between classes
- Makes testing easier with mock dependencies
- Supports flexibility in configuration

## 4. Pipeline Pattern

**Purpose**: Chain processing steps in a sequence, where each step performs a specific transformation.

**Implementation Example**:
```python
class PipelineStep(IPipelineStep):
    """Base class for pipeline steps."""
    
    def __init__(self, strategy: Strategy):
        self._strategy = strategy
    
    @property
    def name(self) -> str:
        """Get the name of this pipeline step."""
        return self._strategy.name
    
    def execute(self, df: pd.DataFrame) -> pd.DataFrame:
        """Process using the strategy."""
        return self._strategy.process(df)


class Pipeline:
    """Pipeline with execution tracking."""
    
    def __init__(self, config_provider: IConfigProvider):
        self._config = config_provider
        self._logger = config_provider.get_logger()
        self._steps: List[ValidationPipelineStep] = []
    
    def add_step(self, step: PipelineStep) -> 'Pipeline':
        """Add a step to the pipeline."""
        self._steps.append(step)
        return self
    
    def process(self, df: pd.DataFrame) -> Dict[str, Any]:
        """Run pipeline with optimized memory usage."""
        # ... initialization
        for step_idx, step in enumerate(self._steps):
            step_result = step.strategy.execute(df)
            # ... process results
        return results  # Return the processed results
```

**Benefits**:
- Sequential processing of data through discrete steps
- Each step has a single responsibility
- Easy to add, remove, or reorder processing steps



## 5. Singleton Pattern
**Purpose**: Ensure a class has only one instance and provide a global point of access to it.

**Implementation Example**:
```python
class Singleton:
    _instance = None

    def __new__(cls, *args, **kwargs):
        if not cls._instance:
            cls._instance = super(Singleton, cls).__new__(cls)
        return cls._instance

    def __init__(self, value=None):
        if not hasattr(self, 'initialized'):
            self.value = value
            self.initialized = True
```
**Benefits**:
- Controlled access to a single instance
- Reduces memory footprint
- Simplifies global state management
- Useful for shared resources like configuration, logging, or connection pools



## 6. Proxy Pattern

**Purpose**: Provide a surrogate or placeholder for another object to control access to it.

**Implementation Example: Configuration Manager**
```python	
class ConfigProvider:
	"""Manages application configuration."""
	
	def __init__(self, config_path: Optional[str] = None, loglevel: Optional[str] = None) -> None:
		"""Initialize the configuration provider."""
		# Set config path relative to script dir if not absolute
		if config_path:
			config_path_obj = Path(config_path)
			if not config_path_obj.is_absolute():
				# If relative path provided, make it relative to script dir
				config_path_obj = Path(__file__).parent / config_path_obj
			self._config_path = config_path_obj
		else:
			# Default config in same directory as this script
			self._config_path = Path(__file__).parent / 'config.yaml'
		
		# Init configuration
		self._config = self._load_default_config()
		self.load_from_file()

		# Init logging
		self._logger = None
		self._initialize_logging(loglevel)
		
	def _initialize_logging(self, loglevel: Optional[str] = None) -> None:
		"""Initialize logging system."""
		log_dir = self._config.get('log_dir', 'Logs')
		if not log_dir.exists():
			log_dir.mkdir(parents=True, exist_ok=True)
			
		log_file = log_dir / f"{datetime.now().strftime('%Y-%m-%d')}_DataFactory.log"
		
		# Configure the logger with the new format including dataset
		logging.basicConfig(
			level=logging.INFO,
			format='%(asctime)s - %(levelname)s - [%(dataset)s] - %(message)s',
			handlers=[
				logging.FileHandler(log_file),
				logging.StreamHandler()
			]
		)
		
		self._logger = logging.getLogger(__name__)

		# Set logger to debug level for detailed output
		if loglevel and loglevel.upper() == 'DEBUG':
			self._logger.setLevel(logging.DEBUG)
		else:
			self._logger.setLevel(logging.INFO)
	
	
	def get_logger(self) -> logging.Logger:
		"""Get logger instance."""
		return self._logger
	
	def _load_default_config(self) -> Dict[str, Any]:
		"""Load default configuration values."""
		root = Path(__file__).parents[1]
		return {
			"OUTPUT_DIR": root / 'Data'
		}
	
	def load_from_file(self, config_path: Optional[str] = None) -> None:
		"""Load configuration from a file."""
		path = config_path or self.config_path
		if not path or not Path(path).exists():
			logging.warning(f"Config file not found: {path}. Using defaults.")
			return
			
		try:
			import yaml
			with open(path, 'r') as f:
				file_config = yaml.safe_load(f)
			
			# Convert string paths to Path objects
			for key, value in file_config.items():
				if isinstance(value, str) and (key.endswith('_DIR') or key.endswith('_PATH')):
					file_config[key] = Path(value)
			
			# Update config
			self.config.update(file_config)
			logging.info(f"Loaded configuration from {path}")

		except Exception as e:
			logging.error(f"Error loading config: {e}")
	
	def get(self, key: str, default: Any = None) -> Any:
		"""Get a configuration value."""
		return self.config.get(key, default)
	
	def get_all(self) -> Dict[str, Any]:
		"""Get all configuration values."""
		return self.config.copy()
```

**Implementation Example: Cache Manager**
```python
class CacheManager(ICacheManager):
    """Manage caching of intermediate results between pipeline stages."""
    
    def get_from_cache(self, dataset_name: str, step_name: str) -> Optional[pd.DataFrame]:
        """Retrieve data from cache if available."""
        if not self._enabled:
            return None
            
        # Generate cache key
        cache_key = f"{dataset_name}_{step_name}"
        
        # Check memory cache first
        if cache_key in self._memory_cache:
            self._logger.info(f"Retrieved {step_name} result from memory cache")
            return self._memory_cache[cache_key].copy()
        
        # Try disk cache
        cache_path = self._cache_dir / f"{cache_key}.pkl"
        if cache_path.exists():
            try:
                df = pd.read_pickle(cache_path)
                # Store in memory cache for future use
                self._memory_cache[cache_key] = df.copy()
                self._logger.info(f"Retrieved {step_name} result from disk cache")
                return df
            except Exception as e:
                self._logger.warning(f"Failed to load cache from {cache_path}: {e}")
        
        return None
```

**Benefits**:
- Controls access to expensive operations
- Implements caching for performance optimization
- Provides transparent access to the real object


## 7. Observer Pattern

**Purpose**: Define a one-to-many dependency so that when one object changes state, all dependents are notified.

**Implementation Example**:
```python
class DatasetFilter(logging.Filter):
    def __init__(self, config):
        super().__init__()
        self.config = config
        self._last_dataset = None
        
    def filter(self, record):
        dataset = self.config.get_config('current_dataset', '')
        record.dataset = dataset
        
        # Track dataset changes for debugging
        if self._last_dataset != dataset:
            self._last_dataset = dataset
        
        return True
```

**Benefits**:
- Implements loose coupling between related components
- Supports state change notifications
- Enables dynamic relationships between objects
