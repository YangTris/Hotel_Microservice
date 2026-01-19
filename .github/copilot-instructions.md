# MyCarterApp - AI Agent Instructions

## Project Overview

This is a **Booking Service** microservice for a Hotel Booking System, built with .NET 9.0 using Clean Architecture principles. This service handles the **booking flow** - managing room reservations, availability, and booking lifecycle.

**Domain Responsibility**: Hotel room booking operations including search availability, create booking, confirm/cancel reservations, and booking status tracking.

## Architecture Pattern

**Clean Architecture + CQRS with Vertical Slice**:

- **API Layer**: Carter modules define HTTP endpoints (presentation)
- **Application Layer**: MediatR handlers implement business logic (CQRS commands/queries)
- **Domain Layer**: Business entities and domain logic
- **Infrastructure Layer**: Marten for data persistence, MassTransit for messaging

**Carter Module Pattern**: Routes are defined by implementing `ICarterModule` - convention-based routing.

- Carter modules call MediatR handlers, not business logic directly
- Auto-discovery: `builder.Services.AddCarter()` and `app.MapCarter()` register all modules
- Minimal `Program.cs`: Framework setup happens through conventions

### Key Conventions

**Carter Module Example** (API/Endpoint Layer):

```csharp
public class BookingModule : ICarterModule
{
    public void AddRoutes(IEndpointRouteBuilder app)
    {
        app.MapPost("/bookings", async (CreateBookingCommand command, IMediator mediator) =>
        {
            var result = await mediator.Send(command);
            return Results.Created($"/bookings/{result.Id}", result);
        });

        app.MapGet("/bookings/{id}", async (Guid id, IMediator mediator) =>
        {
            var result = await mediator.Send(new GetBookingQuery(id));
            return result is not null ? Results.Ok(result) : Results.NotFound();
        });
    }
}
```

**MediatR Handler Example** (Application Layer):

```csharp
public record CreateBookingCommand(Guid RoomId, DateTime CheckIn, DateTime CheckOut) : IRequest<BookingResponse>;

public class CreateBookingHandler : IRequestHandler<CreateBookingCommand, BookingResponse>
{
    private readonly IDocumentSession _session;
    private readonly IPublishEndpoint _publishEndpoint;

    public async Task<BookingResponse> Handle(CreateBookingCommand request, CancellationToken ct)
    {
        var booking = new Booking(request.RoomId, request.CheckIn, request.CheckOut);
        _session.Store(booking);
        await _session.SaveChangesAsync(ct);

        // Publish domain event
        await _publishEndpoint.Publish(new BookingCreatedEvent(booking.Id), ct);

        return new BookingResponse(booking.Id);
    }
}
```

**FluentValidation Example**:

```csharp
public class CreateBookingValidator : AbstractValidator<CreateBookingCommand>
{
    public CreateBookingValidator()
    {
        RuleFor(x => x.RoomId).NotEmpty();
        RuleFor(x => x.CheckIn).GreaterThan(DateTime.UtcNow);
        RuleFor(x => x.CheckOut).GreaterThan(x => x.CheckIn);
    }
}
```

## Development Workflow

### Running the Application

```powershell
dotnet run
```

Default ports:

- HTTP: `http://localhost:5148`
- HTTPS: `https://localhost:7134`

### Building

```powershell
dotnet build
```

### Adding Carter Modules

1. Create a new `.cs` file in `Features/{FeatureName}` folder (vertical slice organization)
2. Implement `ICarterModule` interface
3. Define routes that delegate to MediatR handlers
4. Carter auto-discovers and registers modules - no manual registration

**Example Structure**:

```
Features/
  Bookings/
    CreateBooking/
      CreateBookingCommand.cs       # MediatR command
      CreateBookingHandler.cs       # Command handler
      CreateBookingValidator.cs     # FluentValidation
    GetBooking/
      GetBookingQuery.cs
      GetBookingHandler.cs
    BookingModule.cs                # Carter routes
```

### Configuration

- `appsettings.json` - production settings
- `appsettings.Development.json` - development overrides
- Launch profiles in `Properties/launchSettings.json` define startup behavior

## Technology Stack

- **.NET 9.0** with ASP.NET Core
- **Carter 9.0.0** - Minimal API routing with module discovery
- **MediatR** - CQRS pattern implementation (commands/queries)
- **FluentValidation 11.11.0** - Request validation (auto-integrated via Carter)
- **Marten** - PostgreSQL as a document database and event store
- **MassTransit** - Distributed application framework for message-based communication
- **YARP (Yet Another Reverse Proxy)** - API Gateway (used at system level, not in this service)

### Why These Choices

- **Marten**: Combines document database with event sourcing capabilities, perfect for booking aggregates
- **MassTransit**: Abstracts message brokers (RabbitMQ/Azure Service Bus), enables saga orchestration for booking workflows
- **MediatR**: Decouples API layer from business logic, enables cross-cutting concerns (logging, validation)
- **Carter + Minimal APIs**: Lightweight, convention-based routing without controller overhead
- **FluentValidation**: Expressive validation rules, separates validation from business logic

## Project Settings

- **Nullable reference types enabled**: Use `?` for nullable types, non-nullable by default
- **Implicit usings enabled**: Common namespaces (Microsoft.AspNetCore._, System._, etc.) are auto-imported globally
- **C# 12**: Latest language features available on .NET 9

## Microservice Context

This is the **Booking Service** in a Hotel Microservice architecture. It communicates with:

- **Room Service**: Checks room availability via HTTP or messages
- **Payment Service**: Processes payments via saga orchestration
- **Notification Service**: Sends booking confirmations via events
- **API Gateway (YARP)**: Routes external requests to this service

**Communication Patterns**:

- **Synchronous**: HTTP calls for immediate queries (e.g., check room availability)
- **Asynchronous**: MassTransit events for eventual consistency (e.g., BookingCreated event)
- **Saga Pattern**: Distributed transactions for booking flow (reserve room → process payment → confirm booking)

**Data Ownership**:

- Owns booking aggregates (Booking, Reservation, BookingHistory)
- Stores in PostgreSQL via Marten document database
- Never directly accesses other services' databases

## When Adding Features

1. **New API endpoints**: Create `ICarterModule` implementations, don't modify `Program.cs`
2. **Validation**: Use FluentValidation validators - Carter integrates them automatically
3. **Dependency injection**: Register services in `Program.cs` via `builder.Services.Add*()`
4. **Middleware**: Add to `Program.cs` between `Build()` and `Run()` calls
5. **Database/External services**: Configuration goes in `appsettings.json`, registration in `Program.cs`

## What's NOT Here (Yet)

- No Marten/PostgreSQL configuration
- No MediatR registration
- No MassTransit message bus setup
- No authentication/authorization configured
- No module classes exist yet (this is a fresh Carter template)
- No testing infrastructure set up

When these are added, follow the patterns established in the codebase.
