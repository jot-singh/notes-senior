# LLD Interview Problems - Part 1

## Overview
This document covers common Low-Level Design interview problems with complete solutions. Each problem includes requirements analysis, class design, and implementation.

---

## Parking Lot System

### Requirements

**Functional Requirements:**
- Support multiple floors and different vehicle types
- Track available/occupied spots
- Calculate parking fees based on duration
- Issue tickets and process payments

**Non-Functional Requirements:**
- Handle concurrent entries/exits
- Support different pricing strategies

### Class Design

```
┌─────────────────────────────────────────────────────────────────┐
│                    PARKING LOT SYSTEM                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌───────────────┐    ┌───────────────┐    ┌───────────────┐    │
│  │  ParkingLot   │───>│ ParkingFloor  │───>│ ParkingSpot   │    │
│  └───────────────┘    └───────────────┘    └───────────────┘    │
│         │                                          │             │
│         │                                          │             │
│         ▼                                          ▼             │
│  ┌───────────────┐                        ┌───────────────┐     │
│  │ EntryPanel    │                        │   Vehicle     │     │
│  │ ExitPanel     │                        └───────────────┘     │
│  └───────────────┘                                △             │
│         │                                         │             │
│         ▼                              ┌──────────┼──────────┐  │
│  ┌───────────────┐                     │          │          │  │
│  │ParkingTicket  │                   Car     Motorcycle   Truck │
│  └───────────────┘                                              │
│         │                                                        │
│         ▼                                                        │
│  ┌───────────────┐                                              │
│  │ Payment       │                                              │
│  │ FeeCalculator │                                              │
│  └───────────────┘                                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Implementation

```java
// Enums
public enum VehicleType {
    MOTORCYCLE, CAR, TRUCK
}

public enum SpotType {
    MOTORCYCLE(VehicleType.MOTORCYCLE),
    COMPACT(VehicleType.CAR),
    LARGE(VehicleType.CAR, VehicleType.TRUCK);
    
    private final Set<VehicleType> allowedTypes;
    
    SpotType(VehicleType... types) {
        this.allowedTypes = Set.of(types);
    }
    
    public boolean canFit(VehicleType vehicleType) {
        return allowedTypes.contains(vehicleType);
    }
}

public enum TicketStatus {
    ACTIVE, PAID, LOST
}

// Vehicle hierarchy
public abstract class Vehicle {
    private final String licensePlate;
    private final VehicleType type;
    
    protected Vehicle(String licensePlate, VehicleType type) {
        this.licensePlate = licensePlate;
        this.type = type;
    }
    
    public String getLicensePlate() { return licensePlate; }
    public VehicleType getType() { return type; }
}

public class Car extends Vehicle {
    public Car(String licensePlate) {
        super(licensePlate, VehicleType.CAR);
    }
}

public class Motorcycle extends Vehicle {
    public Motorcycle(String licensePlate) {
        super(licensePlate, VehicleType.MOTORCYCLE);
    }
}

public class Truck extends Vehicle {
    public Truck(String licensePlate) {
        super(licensePlate, VehicleType.TRUCK);
    }
}

// Parking Spot
public class ParkingSpot {
    private final String spotId;
    private final SpotType type;
    private final int floor;
    private Vehicle vehicle;
    private final ReentrantLock lock = new ReentrantLock();
    
    public ParkingSpot(String spotId, SpotType type, int floor) {
        this.spotId = spotId;
        this.type = type;
        this.floor = floor;
    }
    
    public boolean isAvailable() {
        return vehicle == null;
    }
    
    public boolean canFitVehicle(VehicleType vehicleType) {
        return type.canFit(vehicleType);
    }
    
    public boolean assignVehicle(Vehicle vehicle) {
        lock.lock();
        try {
            if (!isAvailable() || !canFitVehicle(vehicle.getType())) {
                return false;
            }
            this.vehicle = vehicle;
            return true;
        } finally {
            lock.unlock();
        }
    }
    
    public Vehicle removeVehicle() {
        lock.lock();
        try {
            Vehicle v = this.vehicle;
            this.vehicle = null;
            return v;
        } finally {
            lock.unlock();
        }
    }
    
    // Getters
    public String getSpotId() { return spotId; }
    public SpotType getType() { return type; }
    public int getFloor() { return floor; }
    public Vehicle getVehicle() { return vehicle; }
}

// Parking Floor
public class ParkingFloor {
    private final int floorNumber;
    private final Map<SpotType, List<ParkingSpot>> spotsByType;
    private final DisplayBoard displayBoard;
    
    public ParkingFloor(int floorNumber, Map<SpotType, Integer> spotCounts) {
        this.floorNumber = floorNumber;
        this.spotsByType = new EnumMap<>(SpotType.class);
        this.displayBoard = new DisplayBoard();
        
        initializeSpots(spotCounts);
    }
    
    private void initializeSpots(Map<SpotType, Integer> spotCounts) {
        int spotCounter = 1;
        for (Map.Entry<SpotType, Integer> entry : spotCounts.entrySet()) {
            List<ParkingSpot> spots = new ArrayList<>();
            for (int i = 0; i < entry.getValue(); i++) {
                String spotId = String.format("F%d-%s-%03d", 
                    floorNumber, entry.getKey().name().charAt(0), spotCounter++);
                spots.add(new ParkingSpot(spotId, entry.getKey(), floorNumber));
            }
            spotsByType.put(entry.getKey(), spots);
        }
        updateDisplayBoard();
    }
    
    public ParkingSpot findAvailableSpot(VehicleType vehicleType) {
        return spotsByType.entrySet().stream()
            .filter(e -> e.getKey().canFit(vehicleType))
            .flatMap(e -> e.getValue().stream())
            .filter(ParkingSpot::isAvailable)
            .findFirst()
            .orElse(null);
    }
    
    public void updateDisplayBoard() {
        Map<SpotType, Long> availableCounts = spotsByType.entrySet().stream()
            .collect(Collectors.toMap(
                Map.Entry::getKey,
                e -> e.getValue().stream().filter(ParkingSpot::isAvailable).count()
            ));
        displayBoard.update(floorNumber, availableCounts);
    }
    
    public int getFloorNumber() { return floorNumber; }
}

// Parking Ticket
public class ParkingTicket {
    private final String ticketId;
    private final Vehicle vehicle;
    private final ParkingSpot spot;
    private final LocalDateTime entryTime;
    private LocalDateTime exitTime;
    private TicketStatus status;
    private BigDecimal fee;
    
    public ParkingTicket(Vehicle vehicle, ParkingSpot spot) {
        this.ticketId = generateTicketId();
        this.vehicle = vehicle;
        this.spot = spot;
        this.entryTime = LocalDateTime.now();
        this.status = TicketStatus.ACTIVE;
    }
    
    private String generateTicketId() {
        return "TKT-" + UUID.randomUUID().toString().substring(0, 8).toUpperCase();
    }
    
    public void markPaid(BigDecimal fee) {
        this.exitTime = LocalDateTime.now();
        this.fee = fee;
        this.status = TicketStatus.PAID;
    }
    
    public Duration getParkingDuration() {
        LocalDateTime end = exitTime != null ? exitTime : LocalDateTime.now();
        return Duration.between(entryTime, end);
    }
    
    // Getters
    public String getTicketId() { return ticketId; }
    public Vehicle getVehicle() { return vehicle; }
    public ParkingSpot getSpot() { return spot; }
    public LocalDateTime getEntryTime() { return entryTime; }
    public TicketStatus getStatus() { return status; }
}

// Fee Calculator (Strategy Pattern)
public interface FeeCalculator {
    BigDecimal calculate(ParkingTicket ticket);
}

public class HourlyFeeCalculator implements FeeCalculator {
    private final Map<VehicleType, BigDecimal> hourlyRates;
    
    public HourlyFeeCalculator() {
        this.hourlyRates = Map.of(
            VehicleType.MOTORCYCLE, new BigDecimal("1.00"),
            VehicleType.CAR, new BigDecimal("2.00"),
            VehicleType.TRUCK, new BigDecimal("4.00")
        );
    }
    
    @Override
    public BigDecimal calculate(ParkingTicket ticket) {
        long hours = Math.max(1, ticket.getParkingDuration().toHours() + 1);
        BigDecimal rate = hourlyRates.get(ticket.getVehicle().getType());
        return rate.multiply(BigDecimal.valueOf(hours));
    }
}

public class FlatRateFeeCalculator implements FeeCalculator {
    private final BigDecimal flatRate;
    
    public FlatRateFeeCalculator(BigDecimal flatRate) {
        this.flatRate = flatRate;
    }
    
    @Override
    public BigDecimal calculate(ParkingTicket ticket) {
        return flatRate;
    }
}

// Payment
public interface PaymentProcessor {
    boolean processPayment(BigDecimal amount, PaymentMethod method);
}

public class ParkingPayment {
    private final PaymentProcessor processor;
    private final FeeCalculator feeCalculator;
    
    public ParkingPayment(PaymentProcessor processor, FeeCalculator feeCalculator) {
        this.processor = processor;
        this.feeCalculator = feeCalculator;
    }
    
    public PaymentResult processTicket(ParkingTicket ticket, PaymentMethod method) {
        BigDecimal fee = feeCalculator.calculate(ticket);
        
        if (processor.processPayment(fee, method)) {
            ticket.markPaid(fee);
            return new PaymentResult(true, fee, "Payment successful");
        }
        return new PaymentResult(false, fee, "Payment failed");
    }
}

// Main Parking Lot class (Singleton)
public class ParkingLot {
    private static volatile ParkingLot instance;
    
    private final String name;
    private final List<ParkingFloor> floors;
    private final Map<String, ParkingTicket> activeTickets;
    private final FeeCalculator feeCalculator;
    private final ReentrantReadWriteLock lock = new ReentrantReadWriteLock();
    
    private ParkingLot(String name, List<ParkingFloor> floors, FeeCalculator feeCalculator) {
        this.name = name;
        this.floors = floors;
        this.activeTickets = new ConcurrentHashMap<>();
        this.feeCalculator = feeCalculator;
    }
    
    public static ParkingLot getInstance(String name, List<ParkingFloor> floors, 
                                         FeeCalculator feeCalculator) {
        if (instance == null) {
            synchronized (ParkingLot.class) {
                if (instance == null) {
                    instance = new ParkingLot(name, floors, feeCalculator);
                }
            }
        }
        return instance;
    }
    
    public ParkingTicket parkVehicle(Vehicle vehicle) {
        lock.writeLock().lock();
        try {
            // Find available spot
            ParkingSpot spot = findAvailableSpot(vehicle.getType());
            if (spot == null) {
                throw new ParkingFullException("No available spots for " + vehicle.getType());
            }
            
            // Assign vehicle to spot
            if (!spot.assignVehicle(vehicle)) {
                throw new ParkingException("Failed to assign spot");
            }
            
            // Create ticket
            ParkingTicket ticket = new ParkingTicket(vehicle, spot);
            activeTickets.put(ticket.getTicketId(), ticket);
            
            // Update display
            floors.get(spot.getFloor() - 1).updateDisplayBoard();
            
            return ticket;
            
        } finally {
            lock.writeLock().unlock();
        }
    }
    
    public BigDecimal exitVehicle(String ticketId, PaymentMethod paymentMethod) {
        lock.writeLock().lock();
        try {
            ParkingTicket ticket = activeTickets.get(ticketId);
            if (ticket == null) {
                throw new InvalidTicketException("Ticket not found: " + ticketId);
            }
            
            // Calculate and process payment
            BigDecimal fee = feeCalculator.calculate(ticket);
            ticket.markPaid(fee);
            
            // Free the spot
            ticket.getSpot().removeVehicle();
            activeTickets.remove(ticketId);
            
            // Update display
            int floorNum = ticket.getSpot().getFloor();
            floors.get(floorNum - 1).updateDisplayBoard();
            
            return fee;
            
        } finally {
            lock.writeLock().unlock();
        }
    }
    
    private ParkingSpot findAvailableSpot(VehicleType vehicleType) {
        for (ParkingFloor floor : floors) {
            ParkingSpot spot = floor.findAvailableSpot(vehicleType);
            if (spot != null) {
                return spot;
            }
        }
        return null;
    }
    
    public int getAvailableSpotCount(VehicleType type) {
        lock.readLock().lock();
        try {
            return floors.stream()
                .mapToInt(f -> (int) f.findAvailableSpot(type) != null ? 1 : 0)
                .sum();
        } finally {
            lock.readLock().unlock();
        }
    }
}

// Usage
public class Main {
    public static void main(String[] args) {
        // Create parking lot
        List<ParkingFloor> floors = List.of(
            new ParkingFloor(1, Map.of(
                SpotType.MOTORCYCLE, 20,
                SpotType.COMPACT, 50,
                SpotType.LARGE, 10
            )),
            new ParkingFloor(2, Map.of(
                SpotType.COMPACT, 80
            ))
        );
        
        ParkingLot lot = ParkingLot.getInstance("Downtown Parking", 
            floors, new HourlyFeeCalculator());
        
        // Park vehicles
        Car car = new Car("ABC-123");
        ParkingTicket ticket = lot.parkVehicle(car);
        System.out.println("Parked at: " + ticket.getSpot().getSpotId());
        
        // Exit
        BigDecimal fee = lot.exitVehicle(ticket.getTicketId(), PaymentMethod.CREDIT_CARD);
        System.out.println("Fee: $" + fee);
    }
}
```

---

## Elevator System

### Requirements

**Functional Requirements:**
- Multiple elevators serving multiple floors
- Handle up/down requests from floors
- Handle destination requests from inside elevator
- Optimal elevator selection algorithm

### Implementation

```java
// Enums
public enum Direction {
    UP, DOWN, IDLE
}

public enum DoorState {
    OPEN, CLOSED
}

// Request
public record ElevatorRequest(int floor, Direction direction, long timestamp) {
    public ElevatorRequest(int floor, Direction direction) {
        this(floor, direction, System.currentTimeMillis());
    }
}

// Elevator
public class Elevator {
    private final int id;
    private final int minFloor;
    private final int maxFloor;
    private int currentFloor;
    private Direction direction;
    private DoorState doorState;
    private final TreeSet<Integer> upStops;
    private final TreeSet<Integer> downStops;
    private final ReentrantLock lock = new ReentrantLock();
    
    public Elevator(int id, int minFloor, int maxFloor) {
        this.id = id;
        this.minFloor = minFloor;
        this.maxFloor = maxFloor;
        this.currentFloor = minFloor;
        this.direction = Direction.IDLE;
        this.doorState = DoorState.CLOSED;
        this.upStops = new TreeSet<>();
        this.downStops = new TreeSet<>(Collections.reverseOrder());
    }
    
    public void addStop(int floor) {
        lock.lock();
        try {
            if (floor < minFloor || floor > maxFloor) {
                throw new IllegalArgumentException("Invalid floor: " + floor);
            }
            
            if (floor > currentFloor) {
                upStops.add(floor);
            } else if (floor < currentFloor) {
                downStops.add(floor);
            }
            
            updateDirection();
        } finally {
            lock.unlock();
        }
    }
    
    public void move() {
        lock.lock();
        try {
            if (direction == Direction.IDLE) {
                return;
            }
            
            // Move one floor
            if (direction == Direction.UP) {
                currentFloor++;
            } else {
                currentFloor--;
            }
            
            // Check if we need to stop
            if (shouldStop()) {
                stop();
            }
            
            updateDirection();
        } finally {
            lock.unlock();
        }
    }
    
    private boolean shouldStop() {
        return (direction == Direction.UP && upStops.contains(currentFloor)) ||
               (direction == Direction.DOWN && downStops.contains(currentFloor));
    }
    
    private void stop() {
        openDoors();
        
        // Remove current floor from stops
        upStops.remove(currentFloor);
        downStops.remove(currentFloor);
        
        // Simulate passenger boarding
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        closeDoors();
    }
    
    private void updateDirection() {
        if (direction == Direction.UP) {
            if (upStops.isEmpty()) {
                direction = downStops.isEmpty() ? Direction.IDLE : Direction.DOWN;
            }
        } else if (direction == Direction.DOWN) {
            if (downStops.isEmpty()) {
                direction = upStops.isEmpty() ? Direction.IDLE : Direction.UP;
            }
        } else {
            // IDLE - pick a direction
            if (!upStops.isEmpty()) {
                direction = Direction.UP;
            } else if (!downStops.isEmpty()) {
                direction = Direction.DOWN;
            }
        }
    }
    
    public void openDoors() {
        doorState = DoorState.OPEN;
        System.out.println("Elevator " + id + ": Doors opening at floor " + currentFloor);
    }
    
    public void closeDoors() {
        doorState = DoorState.CLOSED;
        System.out.println("Elevator " + id + ": Doors closing at floor " + currentFloor);
    }
    
    public int distanceTo(int floor, Direction requestDirection) {
        int distance = Math.abs(currentFloor - floor);
        
        // If moving in same direction and floor is on the way, return direct distance
        if (direction == requestDirection) {
            if (direction == Direction.UP && floor >= currentFloor) {
                return distance;
            }
            if (direction == Direction.DOWN && floor <= currentFloor) {
                return distance;
            }
        }
        
        // If idle, return direct distance
        if (direction == Direction.IDLE) {
            return distance;
        }
        
        // Otherwise, need to complete current trip first
        if (direction == Direction.UP) {
            int topFloor = upStops.isEmpty() ? currentFloor : upStops.last();
            return (topFloor - currentFloor) + (topFloor - floor);
        } else {
            int bottomFloor = downStops.isEmpty() ? currentFloor : downStops.first();
            return (currentFloor - bottomFloor) + (floor - bottomFloor);
        }
    }
    
    // Getters
    public int getId() { return id; }
    public int getCurrentFloor() { return currentFloor; }
    public Direction getDirection() { return direction; }
    public boolean isIdle() { return direction == Direction.IDLE; }
}

// Elevator Selection Strategy
public interface ElevatorSelectionStrategy {
    Elevator selectElevator(List<Elevator> elevators, ElevatorRequest request);
}

public class NearestElevatorStrategy implements ElevatorSelectionStrategy {
    @Override
    public Elevator selectElevator(List<Elevator> elevators, ElevatorRequest request) {
        return elevators.stream()
            .min(Comparator.comparingInt(e -> e.distanceTo(request.floor(), request.direction())))
            .orElseThrow(() -> new NoElevatorAvailableException());
    }
}

// Elevator Controller (Singleton)
public class ElevatorController {
    private static volatile ElevatorController instance;
    
    private final List<Elevator> elevators;
    private final ElevatorSelectionStrategy selectionStrategy;
    private final ExecutorService elevatorExecutor;
    private volatile boolean running;
    
    private ElevatorController(int elevatorCount, int floors, 
                               ElevatorSelectionStrategy strategy) {
        this.elevators = new ArrayList<>();
        for (int i = 0; i < elevatorCount; i++) {
            elevators.add(new Elevator(i + 1, 1, floors));
        }
        this.selectionStrategy = strategy;
        this.elevatorExecutor = Executors.newFixedThreadPool(elevatorCount);
        this.running = true;
        
        startElevators();
    }
    
    public static ElevatorController getInstance(int elevatorCount, int floors) {
        if (instance == null) {
            synchronized (ElevatorController.class) {
                if (instance == null) {
                    instance = new ElevatorController(elevatorCount, floors, 
                        new NearestElevatorStrategy());
                }
            }
        }
        return instance;
    }
    
    private void startElevators() {
        for (Elevator elevator : elevators) {
            elevatorExecutor.submit(() -> {
                while (running) {
                    elevator.move();
                    try {
                        Thread.sleep(1000); // 1 second per floor
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                        break;
                    }
                }
            });
        }
    }
    
    public void requestElevator(int floor, Direction direction) {
        ElevatorRequest request = new ElevatorRequest(floor, direction);
        Elevator elevator = selectionStrategy.selectElevator(elevators, request);
        elevator.addStop(floor);
        System.out.println("Elevator " + elevator.getId() + 
            " assigned to floor " + floor + " (" + direction + ")");
    }
    
    public void selectFloor(int elevatorId, int floor) {
        Elevator elevator = elevators.stream()
            .filter(e -> e.getId() == elevatorId)
            .findFirst()
            .orElseThrow(() -> new IllegalArgumentException("Invalid elevator: " + elevatorId));
        elevator.addStop(floor);
    }
    
    public void shutdown() {
        running = false;
        elevatorExecutor.shutdown();
    }
    
    public void displayStatus() {
        System.out.println("\n=== Elevator Status ===");
        for (Elevator e : elevators) {
            System.out.printf("Elevator %d: Floor %d, Direction: %s%n",
                e.getId(), e.getCurrentFloor(), e.getDirection());
        }
    }
}

// Usage
public class Main {
    public static void main(String[] args) {
        ElevatorController controller = ElevatorController.getInstance(4, 20);
        
        // External requests (from floor buttons)
        controller.requestElevator(5, Direction.UP);
        controller.requestElevator(10, Direction.DOWN);
        controller.requestElevator(1, Direction.UP);
        
        // Internal requests (from inside elevator)
        controller.selectFloor(1, 12);
        controller.selectFloor(2, 7);
        
        controller.displayStatus();
    }
}
```

---

## Vending Machine

### Requirements

**Functional Requirements:**
- Display available products
- Accept multiple payment types
- Handle product selection and dispensing
- Return change
- Inventory management

### Implementation

```java
// Product
public class Product {
    private final String id;
    private final String name;
    private final BigDecimal price;
    
    public Product(String id, String name, BigDecimal price) {
        this.id = id;
        this.name = name;
        this.price = price;
    }
    
    // Getters
    public String getId() { return id; }
    public String getName() { return name; }
    public BigDecimal getPrice() { return price; }
}

// Inventory
public class Inventory {
    private final Map<String, Integer> stock;
    private final Map<String, Product> products;
    
    public Inventory() {
        this.stock = new ConcurrentHashMap<>();
        this.products = new ConcurrentHashMap<>();
    }
    
    public void addProduct(Product product, int quantity) {
        products.put(product.getId(), product);
        stock.merge(product.getId(), quantity, Integer::sum);
    }
    
    public boolean isAvailable(String productId) {
        return stock.getOrDefault(productId, 0) > 0;
    }
    
    public Product getProduct(String productId) {
        return products.get(productId);
    }
    
    public void decrementStock(String productId) {
        stock.computeIfPresent(productId, (k, v) -> v > 0 ? v - 1 : 0);
    }
    
    public Map<Product, Integer> getAllProducts() {
        return products.entrySet().stream()
            .collect(Collectors.toMap(
                Map.Entry::getValue,
                e -> stock.getOrDefault(e.getKey(), 0)
            ));
    }
}

// State Pattern for Vending Machine
public interface VendingMachineState {
    void selectProduct(VendingMachine machine, String productId);
    void insertMoney(VendingMachine machine, BigDecimal amount);
    void dispense(VendingMachine machine);
    void cancel(VendingMachine machine);
}

public class IdleState implements VendingMachineState {
    @Override
    public void selectProduct(VendingMachine machine, String productId) {
        Product product = machine.getInventory().getProduct(productId);
        if (product == null) {
            System.out.println("Invalid product selection");
            return;
        }
        if (!machine.getInventory().isAvailable(productId)) {
            System.out.println("Product out of stock");
            return;
        }
        
        machine.setSelectedProduct(product);
        machine.setState(new HasSelectionState());
        System.out.println("Selected: " + product.getName() + 
            " - Price: $" + product.getPrice());
    }
    
    @Override
    public void insertMoney(VendingMachine machine, BigDecimal amount) {
        System.out.println("Please select a product first");
    }
    
    @Override
    public void dispense(VendingMachine machine) {
        System.out.println("Please select a product first");
    }
    
    @Override
    public void cancel(VendingMachine machine) {
        System.out.println("Nothing to cancel");
    }
}

public class HasSelectionState implements VendingMachineState {
    @Override
    public void selectProduct(VendingMachine machine, String productId) {
        Product product = machine.getInventory().getProduct(productId);
        if (product != null && machine.getInventory().isAvailable(productId)) {
            machine.setSelectedProduct(product);
            System.out.println("Changed selection to: " + product.getName());
        }
    }
    
    @Override
    public void insertMoney(VendingMachine machine, BigDecimal amount) {
        machine.addBalance(amount);
        System.out.println("Inserted: $" + amount + 
            " - Balance: $" + machine.getBalance());
        
        if (machine.getBalance().compareTo(machine.getSelectedProduct().getPrice()) >= 0) {
            machine.setState(new HasMoneyState());
            System.out.println("Sufficient balance. Press dispense button.");
        }
    }
    
    @Override
    public void dispense(VendingMachine machine) {
        System.out.println("Please insert $" + 
            machine.getSelectedProduct().getPrice().subtract(machine.getBalance()) + " more");
    }
    
    @Override
    public void cancel(VendingMachine machine) {
        returnMoney(machine);
        machine.reset();
    }
    
    private void returnMoney(VendingMachine machine) {
        if (machine.getBalance().compareTo(BigDecimal.ZERO) > 0) {
            System.out.println("Returning: $" + machine.getBalance());
            machine.setBalance(BigDecimal.ZERO);
        }
    }
}

public class HasMoneyState implements VendingMachineState {
    @Override
    public void selectProduct(VendingMachine machine, String productId) {
        System.out.println("Please complete current transaction first");
    }
    
    @Override
    public void insertMoney(VendingMachine machine, BigDecimal amount) {
        machine.addBalance(amount);
        System.out.println("Added: $" + amount + " - Balance: $" + machine.getBalance());
    }
    
    @Override
    public void dispense(VendingMachine machine) {
        Product product = machine.getSelectedProduct();
        BigDecimal change = machine.getBalance().subtract(product.getPrice());
        
        machine.setState(new DispensingState());
        
        // Dispense product
        machine.getInventory().decrementStock(product.getId());
        System.out.println("Dispensing: " + product.getName());
        
        // Return change
        if (change.compareTo(BigDecimal.ZERO) > 0) {
            System.out.println("Returning change: $" + change);
        }
        
        machine.reset();
    }
    
    @Override
    public void cancel(VendingMachine machine) {
        System.out.println("Returning: $" + machine.getBalance());
        machine.reset();
    }
}

public class DispensingState implements VendingMachineState {
    @Override
    public void selectProduct(VendingMachine machine, String productId) {
        System.out.println("Please wait, dispensing in progress...");
    }
    
    @Override
    public void insertMoney(VendingMachine machine, BigDecimal amount) {
        System.out.println("Please wait, dispensing in progress...");
    }
    
    @Override
    public void dispense(VendingMachine machine) {
        System.out.println("Already dispensing...");
    }
    
    @Override
    public void cancel(VendingMachine machine) {
        System.out.println("Cannot cancel during dispensing");
    }
}

// Vending Machine
public class VendingMachine {
    private final Inventory inventory;
    private VendingMachineState state;
    private Product selectedProduct;
    private BigDecimal balance;
    
    public VendingMachine() {
        this.inventory = new Inventory();
        this.state = new IdleState();
        this.balance = BigDecimal.ZERO;
    }
    
    // Delegate to state
    public void selectProduct(String productId) {
        state.selectProduct(this, productId);
    }
    
    public void insertMoney(BigDecimal amount) {
        state.insertMoney(this, amount);
    }
    
    public void dispense() {
        state.dispense(this);
    }
    
    public void cancel() {
        state.cancel(this);
    }
    
    public void displayProducts() {
        System.out.println("\n=== Available Products ===");
        inventory.getAllProducts().forEach((product, qty) -> {
            String status = qty > 0 ? "In Stock (" + qty + ")" : "OUT OF STOCK";
            System.out.printf("[%s] %s - $%s - %s%n", 
                product.getId(), product.getName(), product.getPrice(), status);
        });
    }
    
    // Internal methods
    void setState(VendingMachineState state) {
        this.state = state;
    }
    
    void reset() {
        this.selectedProduct = null;
        this.balance = BigDecimal.ZERO;
        this.state = new IdleState();
    }
    
    void addBalance(BigDecimal amount) {
        this.balance = this.balance.add(amount);
    }
    
    // Getters and setters
    Inventory getInventory() { return inventory; }
    Product getSelectedProduct() { return selectedProduct; }
    void setSelectedProduct(Product product) { this.selectedProduct = product; }
    BigDecimal getBalance() { return balance; }
    void setBalance(BigDecimal balance) { this.balance = balance; }
}

// Usage
public class Main {
    public static void main(String[] args) {
        VendingMachine machine = new VendingMachine();
        
        // Stock products
        machine.getInventory().addProduct(
            new Product("A1", "Coca-Cola", new BigDecimal("1.50")), 10);
        machine.getInventory().addProduct(
            new Product("A2", "Pepsi", new BigDecimal("1.50")), 5);
        machine.getInventory().addProduct(
            new Product("B1", "Snickers", new BigDecimal("2.00")), 8);
        machine.getInventory().addProduct(
            new Product("B2", "KitKat", new BigDecimal("1.75")), 0);
        
        machine.displayProducts();
        
        // Transaction flow
        machine.selectProduct("A1");        // Select Coca-Cola
        machine.insertMoney(new BigDecimal("1.00"));  // Insert $1
        machine.insertMoney(new BigDecimal("1.00"));  // Insert another $1
        machine.dispense();                  // Get product and change
        
        machine.displayProducts();
    }
}
```

---

## Related Topics
- [[06-lld-interview-problems-2]] - Chess, Hotel Booking, ATM
- [[01-lld-fundamentals]] - SOLID principles
- [[04-behavioral-patterns]] - State, Strategy patterns
