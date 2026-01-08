# State Pattern & Escrow Logic Explanation

## 1. Escrow/Wallet/Transaction Logic in Code

### Domain Models

```java
public class Wallet {
    private String userId;
    private BigDecimal balance;
    private BigDecimal lockedCredits;
    
    public void lockCredits(BigDecimal amount) {
        if (balance.compareTo(amount) < 0) {
            throw new InsufficientCreditsException("Not enough credits");
        }
        this.balance = this.balance.subtract(amount);
        this.lockedCredits = this.lockedCredits.add(amount);
    }
    
    public void releaseCredits(BigDecimal amount) {
        // Called when session is cancelled - credits return to balance
        this.lockedCredits = this.lockedCredits.subtract(amount);
        this.balance = this.balance.add(amount);
    }
    
    public void transferLockedCredits(BigDecimal amount, Wallet recipient) {
        // Called when session completes - credits go to teacher
        this.lockedCredits = this.lockedCredits.subtract(amount);
        recipient.balance = recipient.balance.add(amount);
    }
}
```

```java
public class EscrowTransaction {
    private String id;
    private String sessionId;
    private String fromWalletId;   // Student's wallet
    private String toWalletId;     // Teacher's wallet (destination on completion)
    private BigDecimal amount;
    private EscrowStatus status;   // HELD, RELEASED, TRANSFERRED, REFUNDED
    private LocalDateTime createdAt;
    private LocalDateTime resolvedAt;
}
```

### Flow: When Student Requests a Session

```java
public class SessionService {
    
    public Session requestSession(String studentId, String listingId, LocalDateTime proposedTime) {
        User student = userRepository.findById(studentId);
        SkillListing listing = listingRepository.findById(listingId);
        Wallet studentWallet = walletRepository.findByUserId(studentId);
        
        // 1. Lock credits in student's wallet
        BigDecimal cost = listing.getCreditCost();
        studentWallet.lockCredits(cost);
        
        // 2. Create escrow transaction record
        EscrowTransaction escrow = new EscrowTransaction();
        escrow.setFromWalletId(studentWallet.getId());
        escrow.setToWalletId(walletRepository.findByUserId(listing.getTeacherId()).getId());
        escrow.setAmount(cost);
        escrow.setStatus(EscrowStatus.HELD);
        escrowRepository.save(escrow);
        
        // 3. Create session with initial state
        Session session = new Session();
        session.setSkillListingId(listingId);
        session.setStudentId(studentId);
        session.setEscrowTransactionId(escrow.getId());
        session.setState(new RequestedState(proposedTime));  // State Pattern
        session.setStateString("REQUESTED");  // Denormalized for queries
        
        sessionRepository.save(session);
        walletRepository.save(studentWallet);
        
        return session;
    }
}
```

---

## 2. What Happens When Teacher Accepts? (State Pattern)

### State Interface

```java
public interface SessionState {
    
    void accept(Session session, SessionContext context);
    void reject(Session session, SessionContext context);
    void counterOffer(Session session, SessionContext context, LocalDateTime newTime);
    void complete(Session session, SessionContext context);
    void dispute(Session session, SessionContext context, String reason);
    
    String getStateName();
}
```

### RequestedState Implementation

```java
public class RequestedState implements SessionState {
    
    private LocalDateTime proposedTime;
    private LocalDateTime expirationTime;
    
    public RequestedState(LocalDateTime proposedTime) {
        this.proposedTime = proposedTime;
        this.expirationTime = proposedTime.plusDays(1); // Auto-expire after 24h
    }
    
    @Override
    public void accept(Session session, SessionContext context) {
        // Teacher accepts the proposed time
        
        // 1. Schedule the session
        session.setScheduledTime(this.proposedTime);
        
        // 2. Transition to AcceptedState
        session.setState(new AcceptedState(this.proposedTime));
        session.setStateString("ACCEPTED");
        
        // 3. Notify the student
        context.getNotificationService().notify(
            session.getStudentId(),
            "Your session request was accepted for " + this.proposedTime
        );
        
        // 4. Escrow remains HELD (credits stay locked until completion)
        // No change to wallet or escrow at this point
    }
    
    @Override
    public void reject(Session session, SessionContext context) {
        // Teacher rejects - refund the student
        
        EscrowTransaction escrow = context.getEscrowRepository()
            .findById(session.getEscrowTransactionId());
        Wallet studentWallet = context.getWalletRepository()
            .findById(escrow.getFromWalletId());
        
        // 1. Release locked credits back to student
        studentWallet.releaseCredits(escrow.getAmount());
        escrow.setStatus(EscrowStatus.REFUNDED);
        
        // 2. Transition to RejectedState (terminal)
        session.setState(new RejectedState());
        session.setStateString("REJECTED");
        
        context.getNotificationService().notify(
            session.getStudentId(),
            "Your session request was rejected"
        );
    }
    
    @Override
    public void counterOffer(Session session, SessionContext context, LocalDateTime newTime) {
        // Teacher proposes different time
        
        // 1. Transition to CounterOfferedState
        session.setState(new CounterOfferedState(newTime, this.proposedTime));
        session.setStateString("COUNTER_OFFERED");
        
        // 2. Escrow remains HELD
        
        context.getNotificationService().notify(
            session.getStudentId(),
            "Teacher proposed alternative time: " + newTime
        );
    }
    
    @Override
    public void complete(Session session, SessionContext context) {
        throw new IllegalStateTransitionException("Cannot complete a session that hasn't been accepted");
    }
    
    @Override
    public void dispute(Session session, SessionContext context, String reason) {
        throw new IllegalStateTransitionException("Cannot dispute a session that hasn't occurred");
    }
    
    @Override
    public String getStateName() {
        return "REQUESTED";
    }
}
```

### AcceptedState Implementation

```java
public class AcceptedState implements SessionState {
    
    private LocalDateTime scheduledTime;
    
    public AcceptedState(LocalDateTime scheduledTime) {
        this.scheduledTime = scheduledTime;
    }
    
    @Override
    public void accept(Session session, SessionContext context) {
        throw new IllegalStateTransitionException("Session already accepted");
    }
    
    @Override
    public void reject(Session session, SessionContext context) {
        throw new IllegalStateTransitionException("Cannot reject an accepted session");
    }
    
    @Override
    public void counterOffer(Session session, SessionContext context, LocalDateTime newTime) {
        throw new IllegalStateTransitionException("Cannot counter-offer an accepted session");
    }
    
    @Override
    public void complete(Session session, SessionContext context) {
        // Session finished successfully - transfer credits to teacher
        
        EscrowTransaction escrow = context.getEscrowRepository()
            .findById(session.getEscrowTransactionId());
        Wallet studentWallet = context.getWalletRepository()
            .findById(escrow.getFromWalletId());
        Wallet teacherWallet = context.getWalletRepository()
            .findById(escrow.getToWalletId());
        
        // 1. Transfer locked credits from student to teacher
        studentWallet.transferLockedCredits(escrow.getAmount(), teacherWallet);
        escrow.setStatus(EscrowStatus.TRANSFERRED);
        escrow.setResolvedAt(LocalDateTime.now());
        
        // 2. Transition to CompletedState (terminal)
        session.setState(new CompletedState());
        session.setStateString("COMPLETED");
        
        // 3. Prompt for reviews
        context.getNotificationService().notify(
            session.getStudentId(),
            "Session completed! Please leave a review."
        );
    }
    
    @Override
    public void dispute(Session session, SessionContext context, String reason) {
        // Something went wrong - admin will investigate
        
        // 1. Escrow remains HELD (frozen until dispute resolved)
        
        // 2. Transition to DisputedState
        session.setState(new DisputedState(reason));
        session.setStateString("DISPUTED");
        
        // 3. Create support ticket
        context.getSupportService().createTicket(session, reason);
    }
    
    @Override
    public String getStateName() {
        return "ACCEPTED";
    }
}
```

### Session Class (Context)

```java
public class Session {
    private String id;
    private String skillListingId;
    private String studentId;
    private String escrowTransactionId;
    private LocalDateTime scheduledTime;
    
    // State Pattern
    private SessionState state;      // Current state object (behavior)
    private String stateString;      // Denormalized for DB queries
    
    // Delegate to current state
    public void accept(SessionContext context) {
        this.state.accept(this, context);
    }
    
    public void reject(SessionContext context) {
        this.state.reject(this, context);
    }
    
    public void counterOffer(SessionContext context, LocalDateTime newTime) {
        this.state.counterOffer(this, context, newTime);
    }
    
    public void complete(SessionContext context) {
        this.state.complete(this, context);
    }
    
    public void dispute(SessionContext context, String reason) {
        this.state.dispute(this, context, reason);
    }
    
    // Setters for state transitions
    public void setState(SessionState state) {
        this.state = state;
    }
    
    public void setStateString(String stateString) {
        this.stateString = stateString;
    }
}
```

---

## 3. State Transition Diagram

![State Transition Diagram](./State%20Transition%20Diagram.drawio.png)
---

## 4. Summary: Accept Flow

When teacher calls `session.accept(context)`:

1. **RequestedState.accept()** is invoked
2. `scheduledTime` is set from `proposedTime`
3. Session transitions to `AcceptedState`
4. `stateString` updated to "ACCEPTED" (for DB queries)
5. Student is notified
6. **Escrow remains HELD** — credits only transfer on `complete()`

The State Pattern ensures:
- Invalid transitions throw exceptions (can't complete a requested session)
- Each state encapsulates its own behavior
- Session delegates to current state object
