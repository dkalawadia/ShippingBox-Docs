# Component Design Specification
## ShippingBox.com
### Version 1.0
### Date: January 19, 2025

## 1. Frontend Components

### 1.1 Application Structure
```typescript
// Directory Structure
src/
├── app/
│   ├── layout/
│   │   ├── Header/
│   │   │   ├── Header.tsx
│   │   │   ├── Navigation.tsx
│   │   │   └── UserMenu.tsx
│   │   ├── Sidebar/
│   │   │   ├── Sidebar.tsx
│   │   │   └── SidebarItem.tsx
│   │   └── Footer/
│   │       └── Footer.tsx
│   │
│   ├── pages/
│   │   ├── calculator/
│   │   │   ├── CalculatorPage.tsx
│   │   │   ├── components/
│   │   │   └── hooks/
│   │   ├── shipping/
│   │   │   ├── ShippingPage.tsx
│   │   │   ├── components/
│   │   │   └── hooks/
│   │   └── account/
│   │       ├── AccountPage.tsx
│   │       ├── components/
│   │       └── hooks/
│   │
│   └── shared/
│       ├── components/
│       │   ├── Button/
│       │   ├── Input/
│       │   ├── Modal/
│       │   └── Form/
│       ├── hooks/
│       │   ├── useApi.ts
│       │   ├── useAuth.ts
│       │   └── useForm.ts
│       └── utils/
│           ├── validation.ts
│           └── formatting.ts
│
├── core/
│   ├── api/
│   │   ├── client.ts
│   │   └── endpoints.ts
│   ├── auth/
│   │   ├── AuthProvider.tsx
│   │   └── useAuth.ts
│   └── store/
│       ├── store.ts
│       ├── reducers/
│       └── actions/
```

### 1.2 Core Components

#### 1.2.1 Calculator Component
```typescript
interface CalculatorProps {
  onCalculate: (result: CalculationResult) => void;
  initialDimensions?: Dimensions;
  mode?: 'simple' | 'advanced';
}

const Calculator: React.FC<CalculatorProps> = ({
  onCalculate,
  initialDimensions,
  mode = 'simple'
}) => {
  const [dimensions, setDimensions] = useState<Dimensions>(initialDimensions ?? {
    length: 0,
    width: 0,
    height: 0
  });

  const [weight, setWeight] = useState<number>(0);
  const [fragility, setFragility] = useState<FragilityLevel>('medium');

  const handleCalculate = async () => {
    const result = await calculateOptimalBox({
      dimensions,
      weight,
      fragility
    });
    onCalculate(result);
  };

  return (
    <div className="calculator-container">
      <DimensionInput
        value={dimensions}
        onChange={setDimensions}
      />
      <WeightInput
        value={weight}
        onChange={setWeight}
      />
      <FragilitySelector
        value={fragility}
        onChange={setFragility}
      />
      <Button onClick={handleCalculate}>
        Calculate
      </Button>
    </div>
  );
};
```

#### 1.2.2 Shipping Rate Component
```typescript
interface ShippingRateProps {
  origin: Address;
  destination: Address;
  package: PackageDetails;
  onRateSelect: (rate: ShippingRate) => void;
}

const ShippingRate: React.FC<ShippingRateProps> = ({
  origin,
  destination,
  package,
  onRateSelect
}) => {
  const [rates, setRates] = useState<ShippingRate[]>([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const fetchRates = async () => {
      setLoading(true);
      try {
        const response = await getRates({
          origin,
          destination,
          package
        });
        setRates(response.rates);
      } catch (err) {
        setError(err.message);
      } finally {
        setLoading(false);
      }
    };

    fetchRates();
  }, [origin, destination, package]);

  return (
    <div className="shipping-rates">
      {loading ? (
        <LoadingSpinner />
      ) : error ? (
        <ErrorMessage message={error} />
      ) : (
        <RateList
          rates={rates}
          onSelect={onRateSelect}
        />
      )}
    </div>
  );
};
```

## 2. Backend Components

### 2.1 Service Interfaces

#### 2.1.1 Calculator Service
```typescript
interface CalculatorService {
  calculateBox(input: BoxCalculationInput): Promise<BoxRecommendation[]>;
  optimizePackaging(items: Item[]): Promise<PackagingStrategy>;
  validateDimensions(dims: Dimensions): ValidationResult;
}

interface BoxCalculationInput {
  dimensions: Dimensions;
  weight: number;
  fragility: FragilityLevel;
  quantity?: number;
  preferences?: PackagingPreferences;
}

interface BoxRecommendation {
  box: Box;
  packingStrategy: PackingStrategy;
  cost: number;
  confidence: number;
}

class BoxCalculatorService implements CalculatorService {
  private readonly boxRepository: BoxRepository;
  private readonly optimizationEngine: OptimizationEngine;

  constructor(
    boxRepository: BoxRepository,
    optimizationEngine: OptimizationEngine
  ) {
    this.boxRepository = boxRepository;
    this.optimizationEngine = optimizationEngine;
  }

  async calculateBox(input: BoxCalculationInput): Promise<BoxRecommendation[]> {
    // Validate input
    this.validateInput(input);

    // Find suitable boxes
    const availableBoxes = await this.boxRepository.findSuitableBoxes({
      minDimensions: this.calculateMinDimensions(input),
      maxWeight: input.weight * 1.2,
      fragility: input.fragility
    });

    // Generate recommendations
    const recommendations = await Promise.all(
      availableBoxes.map(async box => ({
        box,
        packingStrategy: await this.calculatePackingStrategy(input, box),
        cost: await this.calculateTotalCost(box, input),
        confidence: this.calculateConfidenceScore(box, input)
      }))
    );

    return this.sortRecommendations(recommendations);
  }

  private async calculatePackingStrategy(
    input: BoxCalculationInput,
    box: Box
  ): Promise<PackingStrategy> {
    return {
      orientation: await this.optimizationEngine.calculateOptimalOrientation(
        input.dimensions,
        box.dimensions
      ),
      padding: this.calculateRequiredPadding(input.fragility, input.weight),
      weightDistribution: this.calculateWeightDistribution(input.weight, box.dimensions)
    };
  }
}
```

#### 2.1.2 Shipping Service
```typescript
interface ShippingService {
  getRates(request: RateRequest): Promise<RateQuote[]>;
  validateAddress(address: Address): Promise<ValidationResult>;
  calculateDeliveryTime(quote: RateQuote): Promise<DeliveryEstimate>;
}

interface RateRequest {
  origin: Address;
  destination: Address;
  package: PackageDetails;
  preferences?: ShippingPreferences;
}

interface RateQuote {
  carrier: string;
  service: string;
  cost: Money;
  transitTime: TransitTime;
  guaranteedDelivery: boolean;
  restrictions?: string[];
}

class ShippingRateService implements ShippingService {
  private readonly carrierAdapters: Map<CarrierCode, CarrierAdapter>;
  private readonly rateCache: RateCache;
  private readonly addressValidator: AddressValidator;

  async getRates(request: RateRequest): Promise<RateQuote[]> {
    // Validate addresses
    await this.validateAddresses(request.origin, request.destination);

    // Check cache
    const cachedRates = await this.rateCache.get(
      this.generateCacheKey(request)
    );
    if (cachedRates) return cachedRates;

    // Fetch rates from carriers
    const rates = await this.fetchRatesFromCarriers(request);
    
    // Cache results
    await this.cacheRates(request, rates);
    
    return rates;
  }

  private async fetchRatesFromCarriers(request: RateRequest): Promise<RateQuote[]> {
    const ratePromises = Array.from(this.carrierAdapters.entries())
      .map(async ([carrier, adapter]) => {
        try {
          const quote = await adapter.fetchRates(request);
          return { carrier, quote, error: null };
        } catch (error) {
          return { carrier, quote: null, error };
        }
      });

    const results = await Promise.allSettled(ratePromises);
    
    return results
      .filter(result => 
        result.status === 'fulfilled' && result.value.quote !== null
      )
      .map(result => this.processQuote(result.value));
  }
}
```

## 3. Component Communication

### 3.1 Event System
```typescript
interface EventBus {
  publish<T>(event: Event<T>): void;
  subscribe<T>(eventType: string, handler: EventHandler<T>): void;
  unsubscribe<T>(eventType: string, handler: EventHandler<T>): void;
}

class EventBusImpl implements EventBus {
  private handlers: Map<string, EventHandler<any>[]> = new Map();

  publish<T>(event: Event<T>): void {
    const handlers = this.handlers.get(event.type) || [];
    handlers.forEach(handler => handler(event.payload));
  }

  subscribe<T>(eventType: string, handler: EventHandler<T>): void {
    const handlers = this.handlers.get(eventType) || [];
    this.handlers.set(eventType, [...handlers, handler]);
  }
}
```

### 3.2 Service Integration
```typescript
interface ServiceIntegration {
  initialize(): Promise<void>;
  health(): Promise<HealthStatus>;
  metrics(): Promise<ServiceMetrics>;
}

class ServiceIntegrationImpl implements ServiceIntegration {
  private readonly eventBus: EventBus;
  private readonly metrics: MetricsCollector;

  constructor(eventBus: EventBus, metrics: MetricsCollector) {
    this.eventBus = eventBus;
    this.metrics = metrics;
  }

  async initialize(): Promise<void> {
    await this.registerEventHandlers();
    await this.initializeMetrics();
  }
}
```