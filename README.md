# ticket-booking
# Ticket Booking System - Architecture Specification

## 1. Overview
This document describes the design of a ticket booking system. It ensures safe seat reservations, payment integration, and booking confirmation using **Spring Boot services**, **Redis for distributed locking and TTL**, and **Relational Database for persistence and safety**.

---

## 2. High-Level Architecture

**Components:**
- **User/Frontend**
- **Reservation Service** (handles seat reservation with Redis + DB safety)
- **Payment Service** (integrates with payment gateway, ensures reservation validity)
- **Booking Service** (finalizes booking, writes to DB)
- **Redis** (distributed locks + TTL for reservations)
- **Database (SQL)** (source of truth, transaction isolation)
- **Payment Gateway** (external system)

**Data Flow:**
1. User selects seat → Reservation Service reserves (Redis + DB).
2. Payment Service processes payment (calls external gateway).
3. On payment success → Booking Service confirms booking in DB.
4. Reservation auto-expires if not confirmed in time.

---

## 3. Entity-Relationship Model

**Entities:**
- **User** (user_id, name, email)
- **Seat** (seat_id, status [AVAILABLE, RESERVED, BOOKED])
- **Reservation** (reservation_id, seat_id, user_id, reserved_until, status [PENDING, EXPIRED, CONFIRMED])
- **Payment** (payment_id, reservation_id, amount, status [INITIATED, SUCCESS, FAILED])
- **Booking** (booking_id, seat_id, user_id, payment_id, status [CONFIRMED, CANCELLED])

**Relationships:**
- User → Reservation → Seat
- Reservation → Payment → Booking

---

## 4. API Design

| Endpoint | Method | Request Body | Response Body | Status Codes | Description |
|----------|--------|--------------|---------------|--------------|-------------|
| `/reservations` | POST | `{ "userId": 123, "seatId": 456 }` | `{ "reservationId": 789, "seatId": 456, "userId": 123, "status": "RESERVED", "reservedUntil": "2025-08-19T12:35:00Z" }` | 200 OK, 409 Conflict (seat reserved), 500 Server Error | Reserve a seat for a user |
| `/reservations/{reservationId}` | GET | - | `{ "reservationId": 789, "seatId": 456, "userId": 123, "status": "RESERVED", "reservedUntil": "2025-08-19T12:35:00Z" }` | 200 OK, 404 Not Found | Check reservation status |
| `/payments` | POST | `{ "reservationId": 789, "amount": 50.00, "currency": "USD" }` | `{ "paymentId": 321, "reservationId": 789, "status": "PENDING", "gatewayUrl": "https://payment-gateway.com/checkout/321" }` | 200 OK, 400 Bad Request, 500 Server Error | Initiate payment for a reservation |
| `/payments/callback` | POST | `{ "paymentId": 321, "status": "SUCCESS" }` | `{ "paymentId": 321, "reservationId": 789, "status": "SUCCESS" }` | 200 OK, 400 Bad Request | Payment gateway callback to update payment status |
| `/bookings` | POST | `{ "reservationId": 789, "paymentId": 321 }` | `{ "bookingId": 654, "seatId": 456, "userId": 123, "paymentId": 321, "status": "CONFIRMED" }` | 200 OK, 400 Bad Request, 409 Conflict | Confirm booking after payment success |
| `/bookings/{bookingId}` | GET | - | `{ "bookingId": 654, "seatId": 456, "userId": 123, "paymentId": 321, "status": "CONFIRMED" }` | 200 OK, 404 Not Found | Get booking details |

---

## 5. Service Flow (Sequence)

**Reservation Flow:**
1. User requests seat → Reservation Service checks DB.
2. Acquire Redis lock (`lock:seat:{id}`).
3. Mark seat as RESERVED in DB with `reserved_until` timestamp.
4. Store reservation in Redis with TTL.

**Payment Flow:**
1. User calls `/payments` → Payment Service calls external gateway.
2. On success → emits event/webhook.

**Booking Flow:**
1. Booking Service receives payment success.
2. Double-check reservation validity (`reserved_until` not expired).
3. Mark seat as BOOKED in DB.
4. Release Redis lock.

---

## 6. Redis + DB Safety

- **Redis Lock**: Prevents multiple users from reserving the same seat concurrently.
- **DB Safety**: Ensures no double booking using transaction isolation.
- **Isolation Level**: `READ_COMMITTED` or `REPEATABLE_READ` (depending on strictness).
- **TTL**: Reservations expire automatically (e.g., 5 minutes).

---

## 7. Spring Boot Code Snippets

### Redis Lock Utility

@Component
public class RedisLockService {
    @Autowired private StringRedisTemplate redisTemplate;

    public boolean acquireLock(String key, String value, Duration ttl) {
        return Boolean.TRUE.equals(redisTemplate.opsForValue().setIfAbsent(key, value, ttl));
    }

    public void releaseLock(String key, String value) {
        String current = redisTemplate.opsForValue().get(key);
        if (value.equals(current)) {
            redisTemplate.delete(key);
        }
    }
}


@Service
public class ReservationService {
    @Autowired private RedisLockService lockService;
    @Autowired private ReservationRepository reservationRepo;

    @Transactional
    public Reservation reserveSeat(Long seatId, Long userId) {
        String lockKey = "lock:seat:" + seatId;
        String lockVal = UUID.randomUUID().toString();

        if (!lockService.acquireLock(lockKey, lockVal, Duration.ofSeconds(30))) {
            throw new RuntimeException("Seat already locked.");
        }

        try {
            if (reservationRepo.existsActiveReservation(seatId)) {
                throw new RuntimeException("Seat already reserved.");
            }

            Reservation res = new Reservation(seatId, userId, LocalDateTime.now().plusMinutes(5));
            return reservationRepo.save(res);
        } finally {
            lockService.releaseLock(lockKey, lockVal);
        }
    }
}


@Service
public class BookingService {
    @Autowired private ReservationRepository reservationRepo;
    @Autowired private BookingRepository bookingRepo;

    @Transactional
    public Booking confirmBooking(Long reservationId, Long paymentId) {
        Reservation res = reservationRepo.findById(reservationId)
            .orElseThrow(() -> new RuntimeException("Reservation not found"));

        if (res.getReservedUntil().isBefore(LocalDateTime.now())) {
            throw new RuntimeException("Reservation expired.");
        }

        res.setStatus("CONFIRMED");
        reservationRepo.save(res);

        Booking booking = new Booking(res.getSeatId(), res.getUserId(), paymentId, "CONFIRMED");
        return bookingRepo.save(booking);
    }
}
