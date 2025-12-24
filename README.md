# Airdrop_hunter_bot


# Airdrop Hunter Telegram Bot - Complete Project Documentation

## 1. Executive Summary

### 1.1 Project Overview
The Airdrop Hunter Bot is a comprehensive Telegram-based application that aggregates, analyzes, and notifies users about cryptocurrency airdrops. It checks user eligibility, provides claiming instructions, and estimates potential rewards.

### 1.2 Core Features
- Real-time airdrop discovery and aggregation
- Wallet-based eligibility checking
- Personalized notifications and alerts
- Step-by-step claiming instructions
- Reward value estimation
- Historical airdrop tracking
- Community voting and rating system

### 1.3 Technology Stack
- **Backend Framework**: Spring Boot 3.2+
- **Language**: Java 17+
- **Bot Framework**: TelegramBots Java Library
- **Database**: PostgreSQL 15+
- **Cache**: Redis
- **Message Queue**: RabbitMQ
- **Blockchain Integration**: Web3j, Ethers-kt
- **Scheduling**: Spring Scheduler + Quartz
- **API Integration**: RestTemplate, WebClient
- **Security**: Spring Security, JWT

---

## 2. System Architecture

### 2.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     External Services                        │
│  ├─ Telegram API                                            │
│  ├─ Blockchain RPCs (Ethereum, BSC, Polygon, etc.)         │
│  ├─ CoinGecko/CoinMarketCap API                            │
│  ├─ Twitter API / Web Scrapers                             │
│  ├─ Airdrop Aggregator APIs                                │
│  └─ IPFS Gateways                                           │
└─────────────────────────────────────────────────────────────┘
                            ↕
┌─────────────────────────────────────────────────────────────┐
│                    API Gateway Layer                         │
│              (Rate Limiting, Authentication)                 │
└─────────────────────────────────────────────────────────────┘
                            ↕
┌─────────────────────────────────────────────────────────────┐
│                   Application Layer                          │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Telegram Bot Controller                              │  │
│  │  - Command Handlers                                   │  │
│  │  - Callback Query Handlers                            │  │
│  │  - Inline Query Handlers                              │  │
│  └──────────────────────────────────────────────────────┘  │
│                            ↕                                 │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Business Logic Layer                                 │  │
│  │  ├─ Airdrop Discovery Service                        │  │
│  │  ├─ Eligibility Checker Service                      │  │
│  │  ├─ Notification Service                             │  │
│  │  ├─ Wallet Analysis Service                          │  │
│  │  ├─ Value Estimation Service                         │  │
│  │  └─ User Preference Service                          │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                            ↕
┌─────────────────────────────────────────────────────────────┐
│                   Data Access Layer                          │
│  ├─ Repository Layer (JPA/Hibernate)                       │
│  ├─ Cache Layer (Redis)                                    │
│  └─ Message Queue (RabbitMQ)                               │
└─────────────────────────────────────────────────────────────┘
                            ↕
┌─────────────────────────────────────────────────────────────┐
│                   Data Storage Layer                         │
│  ├─ PostgreSQL (Primary Database)                          │
│  └─ Redis (Cache & Session Storage)                        │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Microservices Architecture (Optional for Scaling)

```
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│  Bot Service    │  │ Discovery       │  │  Eligibility    │
│  (Telegram)     │  │ Service         │  │  Service        │
└─────────────────┘  └─────────────────┘  └─────────────────┘
        ↕                    ↕                     ↕
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│ Notification    │  │ Blockchain      │  │  Analytics      │
│ Service         │  │ Service         │  │  Service        │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

---

## 3. Database Schema Design

### 3.1 Entity Relationship Diagram

```
users (1) ─────< (M) user_wallets
  │
  │ (1)
  │
  < (M) user_preferences
  │
  │ (1)
  │
  < (M) user_notifications
  │
  │ (M)
  │
  < (M) user_airdrop_tracking

airdrops (1) ────< (M) airdrop_criteria
  │
  │ (1)
  │
  < (M) airdrop_steps
  │
  │ (1)
  │
  < (M) airdrop_ratings
  │
  │ (M)
  │
  < (M) user_airdrop_tracking

blockchain_networks (1) ────< (M) airdrops
```

### 3.2 Database Tables

#### users
```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    telegram_id BIGINT UNIQUE NOT NULL,
    username VARCHAR(255),
    first_name VARCHAR(255),
    last_name VARCHAR(255),
    language_code VARCHAR(10) DEFAULT 'en',
    is_premium BOOLEAN DEFAULT FALSE,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_interaction TIMESTAMP
);

CREATE INDEX idx_users_telegram_id ON users(telegram_id);
CREATE INDEX idx_users_created_at ON users(created_at);
```

#### user_wallets
```sql
CREATE TABLE user_wallets (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT REFERENCES users(id) ON DELETE CASCADE,
    wallet_address VARCHAR(255) NOT NULL,
    blockchain_network VARCHAR(50) NOT NULL,
    label VARCHAR(255),
    is_primary BOOLEAN DEFAULT FALSE,
    is_verified BOOLEAN DEFAULT FALSE,
    added_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_checked TIMESTAMP,
    UNIQUE(user_id, wallet_address, blockchain_network)
);

CREATE INDEX idx_wallets_user_id ON user_wallets(user_id);
CREATE INDEX idx_wallets_address ON user_wallets(wallet_address);
CREATE INDEX idx_wallets_network ON user_wallets(blockchain_network);
```

#### airdrops
```sql
CREATE TABLE airdrops (
    id BIGSERIAL PRIMARY KEY,
    external_id VARCHAR(255) UNIQUE,
    project_name VARCHAR(255) NOT NULL,
    token_symbol VARCHAR(50),
    token_contract_address VARCHAR(255),
    blockchain_network_id BIGINT REFERENCES blockchain_networks(id),
    description TEXT,
    total_allocation DECIMAL(30, 6),
    estimated_value_usd DECIMAL(15, 2),
    status VARCHAR(50) NOT NULL, -- UPCOMING, ACTIVE, CLAIMING, ENDED, CANCELLED
    start_date TIMESTAMP,
    end_date TIMESTAMP,
    claim_deadline TIMESTAMP,
    official_announcement_url TEXT,
    website_url TEXT,
    twitter_url TEXT,
    discord_url TEXT,
    snapshot_taken BOOLEAN DEFAULT FALSE,
    snapshot_date TIMESTAMP,
    is_verified BOOLEAN DEFAULT FALSE,
    verification_score INTEGER DEFAULT 0,
    risk_level VARCHAR(20), -- LOW, MEDIUM, HIGH, SCAM
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    discovered_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    source VARCHAR(100) -- TWITTER, OFFICIAL, AGGREGATOR, COMMUNITY
);

CREATE INDEX idx_airdrops_status ON airdrops(status);
CREATE INDEX idx_airdrops_network ON airdrops(blockchain_network_id);
CREATE INDEX idx_airdrops_end_date ON airdrops(end_date);
CREATE INDEX idx_airdrops_created_at ON airdrops(created_at);
```

#### airdrop_criteria
```sql
CREATE TABLE airdrop_criteria (
    id BIGSERIAL PRIMARY KEY,
    airdrop_id BIGINT REFERENCES airdrops(id) ON DELETE CASCADE,
    criterion_type VARCHAR(100) NOT NULL, 
    -- Types: MIN_BALANCE, TOKEN_HOLDER, NFT_HOLDER, TRANSACTION_COUNT,
    -- INTERACTION_WITH_CONTRACT, LIQUIDITY_PROVIDER, GOVERNANCE_PARTICIPANT,
    -- TESTNET_USER, SOCIAL_TASK, BRIDGE_USER, etc.
    criterion_value TEXT NOT NULL,
    required_blockchain VARCHAR(50),
    contract_address VARCHAR(255),
    min_amount DECIMAL(30, 6),
    min_transactions INTEGER,
    time_period_start TIMESTAMP,
    time_period_end TIMESTAMP,
    description TEXT,
    is_mandatory BOOLEAN DEFAULT TRUE,
    weight DECIMAL(5, 2) DEFAULT 1.0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_criteria_airdrop_id ON airdrop_criteria(airdrop_id);
CREATE INDEX idx_criteria_type ON airdrop_criteria(criterion_type);
```

#### airdrop_steps
```sql
CREATE TABLE airdrop_steps (
    id BIGSERIAL PRIMARY KEY,
    airdrop_id BIGINT REFERENCES airdrops(id) ON DELETE CASCADE,
    step_number INTEGER NOT NULL,
    step_type VARCHAR(50), -- VERIFICATION, CONNECTION, CLAIM, SOCIAL, TASK
    title VARCHAR(255) NOT NULL,
    description TEXT,
    action_url TEXT,
    is_required BOOLEAN DEFAULT TRUE,
    estimated_time_minutes INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(airdrop_id, step_number)
);

CREATE INDEX idx_steps_airdrop_id ON airdrop_steps(airdrop_id);
```

#### user_airdrop_tracking
```sql
CREATE TABLE user_airdrop_tracking (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT REFERENCES users(id) ON DELETE CASCADE,
    airdrop_id BIGINT REFERENCES airdrops(id) ON DELETE CASCADE,
    is_eligible BOOLEAN,
    eligibility_score DECIMAL(5, 2),
    eligibility_checked_at TIMESTAMP,
    eligibility_details JSONB,
    is_claimed BOOLEAN DEFAULT FALSE,
    claimed_at TIMESTAMP,
    claim_amount DECIMAL(30, 6),
    claim_tx_hash VARCHAR(255),
    is_favorite BOOLEAN DEFAULT FALSE,
    is_notified BOOLEAN DEFAULT FALSE,
    notified_at TIMESTAMP,
    status VARCHAR(50), -- TRACKING, ELIGIBLE, CLAIMED, MISSED, NOT_ELIGIBLE
    notes TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(user_id, airdrop_id)
);

CREATE INDEX idx_tracking_user_id ON user_airdrop_tracking(user_id);
CREATE INDEX idx_tracking_airdrop_id ON user_airdrop_tracking(airdrop_id);
CREATE INDEX idx_tracking_eligible ON user_airdrop_tracking(is_eligible);
CREATE INDEX idx_tracking_status ON user_airdrop_tracking(status);
```

#### user_preferences
```sql
CREATE TABLE user_preferences (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT REFERENCES users(id) ON DELETE CASCADE UNIQUE,
    notification_enabled BOOLEAN DEFAULT TRUE,
    notify_new_airdrops BOOLEAN DEFAULT TRUE,
    notify_eligibility BOOLEAN DEFAULT TRUE,
    notify_claim_reminders BOOLEAN DEFAULT TRUE,
    notify_ending_soon BOOLEAN DEFAULT TRUE,
    min_estimated_value_usd DECIMAL(10, 2) DEFAULT 0,
    preferred_networks TEXT[], -- Array of blockchain networks
    risk_tolerance VARCHAR(20) DEFAULT 'MEDIUM', -- LOW, MEDIUM, HIGH
    exclude_social_tasks BOOLEAN DEFAULT FALSE,
    notification_frequency VARCHAR(20) DEFAULT 'INSTANT', -- INSTANT, DAILY, WEEKLY
    preferred_timezone VARCHAR(50) DEFAULT 'UTC',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_preferences_user_id ON user_preferences(user_id);
```

#### blockchain_networks
```sql
CREATE TABLE blockchain_networks (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) UNIQUE NOT NULL,
    chain_id INTEGER UNIQUE,
    rpc_url TEXT NOT NULL,
    explorer_url TEXT,
    symbol VARCHAR(10),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Pre-populate common networks
INSERT INTO blockchain_networks (name, chain_id, rpc_url, explorer_url, symbol) VALUES
('Ethereum', 1, 'https://eth-mainnet.g.alchemy.com/v2/', 'https://etherscan.io', 'ETH'),
('BSC', 56, 'https://bsc-dataseed.binance.org/', 'https://bscscan.com', 'BNB'),
('Polygon', 137, 'https://polygon-rpc.com/', 'https://polygonscan.com', 'MATIC'),
('Arbitrum', 42161, 'https://arb1.arbitrum.io/rpc', 'https://arbiscan.io', 'ETH'),
('Optimism', 10, 'https://mainnet.optimism.io', 'https://optimistic.etherscan.io', 'ETH');
```

#### airdrop_ratings
```sql
CREATE TABLE airdrop_ratings (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT REFERENCES users(id) ON DELETE CASCADE,
    airdrop_id BIGINT REFERENCES airdrops(id) ON DELETE CASCADE,
    rating INTEGER CHECK (rating >= 1 AND rating <= 5),
    comment TEXT,
    is_scam_report BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(user_id, airdrop_id)
);

CREATE INDEX idx_ratings_airdrop_id ON airdrop_ratings(airdrop_id);
```

#### user_notifications
```sql
CREATE TABLE user_notifications (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT REFERENCES users(id) ON DELETE CASCADE,
    airdrop_id BIGINT REFERENCES airdrops(id) ON DELETE SET NULL,
    notification_type VARCHAR(50) NOT NULL,
    title VARCHAR(255),
    message TEXT NOT NULL,
    is_read BOOLEAN DEFAULT FALSE,
    sent_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    read_at TIMESTAMP
);

CREATE INDEX idx_notifications_user_id ON user_notifications(user_id);
CREATE INDEX idx_notifications_sent_at ON user_notifications(sent_at);
```

---

## 4. Core Algorithm Components

### 4.1 Airdrop Discovery Algorithm

```java
/**
 * Discovers new airdrops from multiple sources
 * Priority: Official Announcements > Aggregators > Social Media > Community
 */
@Service
@Slf4j
public class AirdropDiscoveryService {
    
    private final List<AirdropSource> sources;
    private final AirdropRepository airdropRepository;
    private final AirdropValidator airdropValidator;
    private final DuplicateDetector duplicateDetector;
    
    @Scheduled(fixedDelay = 300000) // Every 5 minutes
    public void discoverNewAirdrops() {
        log.info("Starting airdrop discovery cycle");
        
        List<AirdropCandidate> candidates = new ArrayList<>();
        
        // Step 1: Collect from all sources in parallel
        List<CompletableFuture<List<AirdropCandidate>>> futures = sources.stream()
            .map(source -> CompletableFuture.supplyAsync(() -> {
                try {
                    return source.fetchAirdrops();
                } catch (Exception e) {
                    log.error("Error fetching from source: {}", source.getName(), e);
                    return Collections.emptyList();
                }
            }))
            .collect(Collectors.toList());
        
        candidates = futures.stream()
            .map(CompletableFuture::join)
            .flatMap(List::stream)
            .collect(Collectors.toList());
        
        // Step 2: Remove duplicates using similarity algorithm
        List<AirdropCandidate> uniqueCandidates = duplicateDetector
            .removeDuplicates(candidates);
        
        // Step 3: Validate and score each candidate
        List<ValidatedAirdrop> validatedAirdrops = uniqueCandidates.stream()
            .map(candidate -> airdropValidator.validate(candidate))
            .filter(ValidatedAirdrop::isValid)
            .collect(Collectors.toList());
        
        // Step 4: Enrich with additional data
        validatedAirdrops.forEach(this::enrichAirdropData);
        
        // Step 5: Save to database
        validatedAirdrops.forEach(this::saveAirdrop);
        
        log.info("Discovery completed. Found {} new airdrops", validatedAirdrops.size());
    }
    
    private void enrichAirdropData(ValidatedAirdrop airdrop) {
        // Fetch token price if available
        if (airdrop.hasTokenContract()) {
            airdrop.setEstimatedValueUsd(
                priceService.estimateTokenValue(airdrop.getTokenContract())
            );
        }
        
        // Check social media presence
        airdrop.setSocialScore(
            socialAnalyzer.analyzeSocialPresence(airdrop)
        );
        
        // Analyze smart contract if available
        if (airdrop.hasSmartContract()) {
            ContractAnalysis analysis = contractAnalyzer
                .analyze(airdrop.getContractAddress());
            airdrop.setRiskLevel(analysis.getRiskLevel());
        }
    }
    
    private void saveAirdrop(ValidatedAirdrop validatedAirdrop) {
        Airdrop airdrop = validatedAirdrop.toEntity();
        
        // Check if already exists
        Optional<Airdrop> existing = airdropRepository
            .findByExternalId(airdrop.getExternalId());
        
        if (existing.isPresent()) {
            // Update existing
            updateExistingAirdrop(existing.get(), airdrop);
        } else {
            // Save new
            airdropRepository.save(airdrop);
            notificationService.notifyNewAirdrop(airdrop);
        }
    }
}
```

### 4.2 Eligibility Checking Algorithm

```java
/**
 * Multi-layered eligibility checking system
 * Checks wallet against multiple criteria in parallel
 */
@Service
@Slf4j
public class EligibilityCheckerService {
    
    private final Web3Service web3Service;
    private final CriteriaEvaluatorFactory evaluatorFactory;
    private final EligibilityScoreCalculator scoreCalculator;
    
    public EligibilityResult checkEligibility(User user, Airdrop airdrop) {
        List<UserWallet> wallets = user.getWallets();
        
        if (wallets.isEmpty()) {
            return EligibilityResult.noWallets();
        }
        
        // Get all criteria for this airdrop
        List<AirdropCriterion> criteria = airdrop.getCriteria();
        
        // Check each wallet against all criteria in parallel
        List<WalletEligibility> walletResults = wallets.parallelStream()
            .map(wallet -> checkWalletEligibility(wallet, criteria, airdrop))
            .collect(Collectors.toList());
        
        // Find the best eligible wallet
        Optional<WalletEligibility> bestWallet = walletResults.stream()
            .filter(WalletEligibility::isEligible)
            .max(Comparator.comparing(WalletEligibility::getScore));
        
        if (bestWallet.isPresent()) {
            return EligibilityResult.eligible(
                bestWallet.get(),
                calculateEstimatedReward(bestWallet.get(), airdrop)
            );
        }
        
        // Return most promising wallet even if not eligible
        WalletEligibility mostPromising = walletResults.stream()
            .max(Comparator.comparing(WalletEligibility::getScore))
            .orElse(null);
        
        return EligibilityResult.notEligible(mostPromising);
    }
    
    private WalletEligibility checkWalletEligibility(
            UserWallet wallet, 
            List<AirdropCriterion> criteria,
            Airdrop airdrop) {
        
        WalletEligibility result = new WalletEligibility(wallet);
        
        // Evaluate each criterion
        List<CriterionResult> criterionResults = criteria.parallelStream()
            .map(criterion -> evaluateCriterion(wallet, criterion, airdrop))
            .collect(Collectors.toList());
        
        result.setCriterionResults(criterionResults);
        
        // Calculate overall eligibility
        boolean isEligible = checkMandatoryCriteria(criterionResults) &&
                            checkMinimumScore(criterionResults);
        
        result.setEligible(isEligible);
        result.setScore(scoreCalculator.calculate(criterionResults));
        
        return result;
    }
    
    private CriterionResult evaluateCriterion(
            UserWallet wallet,
            AirdropCriterion criterion,
            Airdrop airdrop) {
        
        CriteriaEvaluator evaluator = evaluatorFactory
            .getEvaluator(criterion.getCriterionType());
        
        try {
            return evaluator.evaluate(wallet, criterion, airdrop);
        } catch (Exception e) {
            log.error("Error evaluating criterion {} for wallet {}", 
                     criterion.getId(), wallet.getWalletAddress(), e);
            return CriterionResult.error(criterion, e.getMessage());
        }
    }
    
    private boolean checkMandatoryCriteria(List<CriterionResult> results) {
        return results.stream()
            .filter(r -> r.getCriterion().isMandatory())
            .allMatch(CriterionResult::isPassed);
    }
    
    private boolean checkMinimumScore(List<CriterionResult> results) {
        double totalScore = results.stream()
            .mapToDouble(r -> r.getScore() * r.getCriterion().getWeight())
            .sum();
        
        double totalWeight = results.stream()
            .mapToDouble(r -> r.getCriterion().getWeight())
            .sum();
        
        return (totalScore / totalWeight) >= 0.5; // 50% threshold
    }
    
    private BigDecimal calculateEstimatedReward(
            WalletEligibility eligibility,
            Airdrop airdrop) {
        
        if (airdrop.getTotalAllocation() == null) {
            return BigDecimal.ZERO;
        }
        
        // Simple proportional calculation based on score
        // More sophisticated algorithms can be implemented
        double scoreRatio = eligibility.getScore() / 100.0;
        
        return airdrop.getTotalAllocation()
            .multiply(BigDecimal.valueOf(scoreRatio))
            .multiply(BigDecimal.valueOf(0.001)); // Assume 0.1% per eligible user
    }
}
```

### 4.3 Specific Criterion Evaluators

```java
/**
 * Token Balance Criterion Evaluator
 */
@Component
public class TokenBalanceEvaluator implements CriteriaEvaluator {
    
    private final Web3Service web3Service;
    
    @Override
    public CriterionResult evaluate(
            UserWallet wallet,
            AirdropCriterion criterion,
            Airdrop airdrop) {
        
        String tokenContract = criterion.getContractAddress();
        BigDecimal minBalance = criterion.getMinAmount();
        
        // Get token balance at snapshot time or current
        BigDecimal balance = web3Service.getTokenBalance(
            wallet.getWalletAddress(),
            tokenContract,
            airdrop.getSnapshotDate()
        );
        
        boolean passed = balance.compareTo(minBalance) >= 0;
        double score = calculateBalanceScore(balance, minBalance);
        
        return CriterionResult.builder()
            .criterion(criterion)
            .passed(passed)
            .score(score)
            .details(Map.of(
                "balance", balance.toString(),
                "required", minBalance.toString(),
                "hasEnough", passed
            ))
            .build();
    }
    
    private double calculateBalanceScore(BigDecimal balance, BigDecimal required) {
        if (balance.compareTo(required) < 0) {
            return balance.divide(required, 2, RoundingMode.HALF_UP)
                .multiply(BigDecimal.valueOf(50))
                .doubleValue();
        }
        
        // Over-qualified: cap at 100
        return Math.min(100.0, 
            50.0 + (balance.divide(required, 2, RoundingMode.HALF_UP)
                .doubleValue() * 10));
    }
}

/**
 * Transaction Count Evaluator
 */
@Component
public class TransactionCountEvaluator implements CriteriaEvaluator {
    
    private final BlockchainExplorerService explorerService;
    
    @Override
    public CriterionResult evaluate(
            UserWallet wallet,
            AirdropCriterion criterion,
            Airdrop airdrop) {
        
        int minTransactions = criterion.getMinTransactions();
        LocalDateTime startDate = criterion.getTimePeriodStart();
        LocalDateTime endDate = criterion.getTimePeriodEnd();
        
        // Get transaction count in time period
        int txCount = explorerService.getTransactionCount(
            wallet.getWalletAddress(),
            wallet.getBlockchainNetwork(),
            startDate,
            endDate
        );
        
        boolean passed = txCount >= minTransactions;
        double score = calculateTxScore(txCount, minTransactions);
        
        return CriterionResult.builder()
            .criterion(criterion)
            .passed(passed)
            .score(score)
            .details(Map.of(
                "transactionCount", txCount,
                "required", minTransactions,
                "period", startDate + " to " + endDate
            ))
            .build();
    }
    
    private double calculateTxScore(int actual, int required) {
        if (actual < required) {
            return (double) actual / required * 50.0;
        }
        return Math.min(100.0, 50.0 + Math.log10(actual - required + 1) * 10);
    }
}

/**
 * NFT Holder Evaluator
 */
@Component
public class NFTHolderEvaluator implements CriteriaEvaluator {
    
    private final NFTService nftService;
    
    @Override
    public CriterionResult evaluate(
            UserWallet wallet,
            AirdropCriterion criterion,
            Airdrop airdrop) {
        
        String nftContract = criterion.getContractAddress();
        
        List<NFT> nfts = nftService.getNFTsOwnedByWallet(
            wallet.getWalletAddress(),
            nftContract,
            airdrop.getSnapshotDate()
        );
        
        boolean passed = !nfts.isEmpty();
        double score = calculateNFTScore(nfts.size());
        
        return CriterionResult.builder()
            .criterion(criterion)
            .passed(passed)
            .score(score)
            .details(Map.of(
                "nftsOwned", nfts.size(),
                "nftContract", nftContract,
                "tokenIds", nfts.stream()
                    .map(NFT::getTokenId)
                    .collect(Collectors.toList())
            ))
            .build();
    }
    
    private double calculateNFTScore(int nftCount) {
        if (nftCount == 0) return 0.0;
        return Math.min(100.0, 50.0 + Math.log10(nftCount) * 20);
    }
}

/**
 * Contract Interaction Evaluator
 */
@Component
public class ContractInteractionEvaluator implements CriteriaEvaluator {
    
    private final Web3Service web3Service;
    
    @Override
    public CriterionResult evaluate(
            UserWallet wallet,
            AirdropCriterion criterion,
            Airdrop airdrop) {
        
        String contractAddress = criterion.getContractAddress();
        LocalDateTime startDate = criterion.getTimePeriodStart();
        LocalDateTime endDate = criterion.getTimePeriodEnd();
        
        List<Transaction> interactions = web3Service.getContractInteractions(
            wallet.getWalletAddress(),
            contractAddress,
            startDate,
            endDate
        );
        
        boolean passed = !interactions.isEmpty();
        double score = calculateInteractionScore(interactions);
        
        return CriterionResult.builder()
            .criterion(criterion)
            .passed(passed)
            .score(score)
            .details(Map.of(
                "interactionCount", interactions.size(),
                "contractAddress", contractAddress,
                "firstInteraction", interactions.isEmpty() ? null : 
                    interactions.get(0).getTimestamp(),
                "lastInteraction", interactions.isEmpty() ? null :
                    interactions.get(interactions.size() - 1).getTimestamp()
            ))
            .build();
    }
    
    private double calculateInteractionScore(List<Transaction> interactions) {
        if (interactions.isEmpty()) return 0.0;
        
        int count = interactions.size();
        long daySpan = calculateDaySpan(interactions);
        
        double countScore = Math.min(50.0, count * 5.0);
        double consistencyScore = Math.min(50.0, daySpan / 7.0 * 10);
        
        return countScore + consistencyScore;
    }
    
    private long calculateDaySpan(List<Transaction> txs) {
        if (txs.size() < 2) return
