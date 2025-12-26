
- **[[Optimistic Locking]]** is ideal when **conflicts are rare** because it maximizes concurrency, and the penalty of a failed transaction is low.
    
- [[**Pessimistic Locking]]** is ideal when **conflicts are frequent** because it minimizes wasted work and provides a clearer, guaranteed mechanism for sequencing updates, though at the cost of reduced concurrency.

## [[Two Phase Commit]] [[2PC]] ##

Needed when the data required for a trasaction is across more than one node. Your service or one of the nodes become a co-ordinator.
> co-ordinator sends prepare command to all participant DB nodes- and keeps the progress WAL-ed in its disk in case it crashes at any point in middle of it.
> all participant db nodes act on that and internally prepare the record. As part of this they do two things, update the durable WAL to have new transaction prepare status for given TID and also update InternalTransaction table  (in memory but having WAL own to protect against crash). At this point these transactions are in *Open* state and are called Open Transactions
> once all nodes have confirmed, co-ordinator asks nodes to commit
> nodes lookup in their InternalTransactionTable and commit the data and let the co-ordinator know.

![[Screenshot 2025-10-30 at 6.53.00 AM.png]]

Serializable isolation over pessimistic locking: when you dont know what to lock but you have to keep systems in a good state across transactions- e.g. only 10 vendors should be available at after 9th, no two concurrent Tx should be able to create 1 each leading to total of 11. 


#### Connection/TxId managements
DB typically internally stores each connectionId with an activeTransactionId and assumes that subsequent steps in a multi-step transaction over same connection belong to that original transaction. 


```
import java.sql.*;

public class MoneyTransferService {

    // Simulating a database connection pool lookup
    private Connection getConnection() throws SQLException {
        // In a real application, this would come from a DataSource/Pool
        return DriverManager.getConnection("jdbc:postgresql://localhost:5432/mydb", "user", "password");
    }

    public boolean transferFunds(String fromUser, String toUser, double amount) {
        Connection conn = null;
        try {
            // STEP 1: START THE TRANSACTION
            conn = getConnection();
            // Tell the database: DO NOT auto-commit after every statement.
            // The transaction is now OPEN.
            conn.setAutoCommit(false); 
            
            // --- Database Interaction 1: Lock and Debit ---
            
            // This is the CRITICAL pessimistic lock (SELECT FOR UPDATE)
            // It holds the connection open and locks Alice's row.
            String selectFromSql = "SELECT balance FROM accounts WHERE user_id = ? FOR UPDATE";
            try (PreparedStatement stmt = conn.prepareStatement(selectFromSql)) {
                stmt.setString(1, fromUser);
                try (ResultSet rs = stmt.executeQuery()) {
                    if (!rs.next()) {
                        throw new SQLException("Source account not found.");
                    }
                    double fromBalance = rs.getDouble("balance");
                    
                    // APPLICATION BUSINESS LOGIC (Executed on the Java side)
                    if (fromBalance < amount) {
                        System.out.println("Insufficient funds. Initiating rollback.");
                        conn.rollback(); // Transaction ABORTED
                        return false;
                    }
                }
            }
            
            // --- Database Interaction 2: Perform the Update ---
            // This runs on the same OPEN transaction, using the previously acquired lock.
            String updateFromSql = "UPDATE accounts SET balance = balance - ? WHERE user_id = ?";
            try (PreparedStatement stmt = conn.prepareStatement(updateFromSql)) {
                stmt.setDouble(1, amount);
                stmt.setString(2, fromUser);
                stmt.executeUpdate(); // DB changes logged, but not yet permanent.
            }

            // --- Database Interaction 3: Credit the second account ---
            String updateToSql = "UPDATE accounts SET balance = balance + ? WHERE user_id = ?";
            try (PreparedStatement stmt = conn.prepareStatement(updateToSql)) {
                stmt.setDouble(1, amount);
                stmt.setString(2, toUser);
                stmt.executeUpdate(); // DB changes logged, but not yet permanent.
            }

            // STEP 2: COMMIT THE TRANSACTION
            // This single call sends the final COMMIT command to the database.
            // All changes from the two UPDATES become permanent and visible.
            // All locks (including the one from FOR UPDATE) are released.
            conn.commit(); 
            System.out.println("Transfer successful and committed.");
            return true;

        } catch (SQLException e) {
            // Handle any database error (e.g., connection lost, deadlock)
            System.err.println("Transaction failed: " + e.getMessage());
            if (conn != null) {
                try {
                    // STEP 2 (Alternative): ROLLBACK THE TRANSACTION
                    // All previous changes (debit and credit) are undone.
                    conn.rollback(); 
                    System.out.println("Transaction successfully rolled back.");
                } catch (SQLException rollbackEx) {
                    System.err.println("Rollback failed: " + rollbackEx.getMessage());
                }
            }
            return false;
        } finally {
            // Important: Close the connection to release resources.
            if (conn != null) {
                try {
                    conn.close();
                } catch (SQLException closeEx) {
                    System.err.println("Could not close connection: " + closeEx.getMessage());
                }
            }
        }
    }
}
```