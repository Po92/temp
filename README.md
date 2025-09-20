// Updated Database Configuration to support multiple packages
@Configuration
public class MultiPackageDatabaseConfiguration {

    @Autowired
    private MultiDatabaseProperties databaseProperties;

    // Main dynamic datasource bean
    @Bean
    @Primary
    public DataSource dataSource() {
        DynamicDataSource dynamicDataSource = new DynamicDataSource();
        dynamicDataSource.setTargetDataSources(createTargetDataSources());
        dynamicDataSource.setDefaultTargetDataSource(createDefaultDataSource());
        dynamicDataSource.afterPropertiesSet();
        return dynamicDataSource;
    }

    private Map<Object, Object> createTargetDataSources() {
        Map<Object, Object> targetDataSources = new HashMap<>();
        
        databaseProperties.getDatasource().forEach((prodLine, environments) -> {
            environments.forEach((environment, databases) -> {
                databases.forEach((database, config) -> {
                    String key = String.format("%s-%s-%s", prodLine, environment, database);
                    DataSource dataSource = createDataSource(config);
                    targetDataSources.put(key, dataSource);
                });
            });
        });
        
        return targetDataSources;
    }
    
    private DataSource createDataSource(DatabaseConfig config) {
        HikariDataSource dataSource = new HikariDataSource();
        dataSource.setJdbcUrl(config.getJdbcUrl());
        dataSource.setUsername(config.getUsername());
        dataSource.setPassword(config.getPassword());
        dataSource.setDriverClassName(config.getDriverClassName());
        dataSource.setMaximumPoolSize(config.getMaximumPoolSize() != null ? config.getMaximumPoolSize() : 10);
        dataSource.setMinimumIdle(config.getMinimumIdle() != null ? config.getMinimumIdle() : 5);
        dataSource.setConnectionTimeout(config.getConnectionTimeout() != null ? config.getConnectionTimeout() : 30000);
        dataSource.setIdleTimeout(config.getIdleTimeout() != null ? config.getIdleTimeout() : 600000);
        dataSource.setMaxLifetime(config.getMaxLifetime() != null ? config.getMaxLifetime() : 1800000);
        return dataSource;
    }
    
    private DataSource createDefaultDataSource() {
        return createDataSource(databaseProperties.getDatasource()
            .values().iterator().next()
            .values().iterator().next()
            .values().iterator().next());
    }
}

// ====================================================================
// PTCLadies Package Structure
// ====================================================================

// PTCLadies Configuration (com.yourapp.ptcladies.config)
@Configuration
@EnableJpaRepositories(
    basePackages = "com.yourapp.ptcladies.repository",
    entityManagerFactoryRef = "ptcladiesEntityManagerFactory",
    transactionManagerRef = "ptcladiesTransactionManager"
)
@EntityScan("com.yourapp.ptcladies.entity")
public class PTCLadiesJpaConfiguration {

    @Bean(name = "ptcladiesEntityManagerFactory")
    public LocalContainerEntityManagerFactoryBean ptcladiesEntityManagerFactory(
            @Qualifier("dataSource") DataSource dataSource) {
        
        LocalContainerEntityManagerFactoryBean em = new LocalContainerEntityManagerFactoryBean();
        em.setDataSource(dataSource);
        em.setPackagesToScan("com.yourapp.ptcladies.entity");
        em.setPersistenceUnitName("ptcladies");
        
        HibernateJpaVendorAdapter vendorAdapter = new HibernateJpaVendorAdapter();
        em.setJpaVendorAdapter(vendorAdapter);
        
        Properties props = new Properties();
        props.setProperty("hibernate.hbm2ddl.auto", "none");
        props.setProperty("hibernate.dialect", "org.hibernate.dialect.SQLServerDialect");
        em.setJpaProperties(props);
        
        return em;
    }

    @Bean(name = "ptcladiesTransactionManager")
    public PlatformTransactionManager ptcladiesTransactionManager(
            @Qualifier("ptcladiesEntityManagerFactory") EntityManagerFactory entityManagerFactory) {
        return new JpaTransactionManager(entityManagerFactory);
    }
}

// PTCLadies Entities (com.yourapp.ptcladies.entity)
@Entity
@Table(name = "ws_product")
public class PTCLadiesWsProduct {
    @Id
    @Column(name = "product_id")
    private String productId;
    
    @Column(name = "product_name")
    private String productName;
    
    @Column(name = "param_id")
    private String paramId;
    
    // Getters and setters
    public String getProductId() { return productId; }
    public void setProductId(String productId) { this.productId = productId; }
    
    public String getProductName() { return productName; }
    public void setProductName(String productName) { this.productName = productName; }
    
    public String getParamId() { return paramId; }
    public void setParamId(String paramId) { this.paramId = paramId; }
}

@Entity
@Table(name = "param_setting")
public class PTCLadiesParamSetting {
    @Id
    @Column(name = "param_id")
    private String paramId;
    
    @Column(name = "param_name")
    private String paramName;
    
    @Column(name = "param_value")
    private String paramValue;
    
    // Getters and setters
    public String getParamId() { return paramId; }
    public void setParamId(String paramId) { this.paramId = paramId; }
    
    public String getParamName() { return paramName; }
    public void setParamName(String paramName) { this.paramName = paramName; }
    
    public String getParamValue() { return paramValue; }
    public void setParamValue(String paramValue) { this.paramValue = paramValue; }
}

// PTCLadies Repository (com.yourapp.ptcladies.repository)
@Repository
public interface PTCLadiesWsProductRepository extends JpaRepository<PTCLadiesWsProduct, String> {
    
    @Query("SELECT new com.yourapp.ptcladies.dto.PTCLadiesProductDto(p.productId, p.productName, ps.paramName, ps.paramValue) " +
           "FROM PTCLadiesWsProduct p JOIN PTCLadiesParamSetting ps ON p.paramId = ps.paramId")
    List<PTCLadiesProductDto> findProductsWithParams();
    
    @Query("SELECT new com.yourapp.ptcladies.dto.PTCLadiesProductDto(p.productId, p.productName, ps.paramName, ps.paramValue) " +
           "FROM PTCLadiesWsProduct p JOIN PTCLadiesParamSetting ps ON p.paramId = ps.paramId " +
           "WHERE p.productName LIKE %:productName%")
    List<PTCLadiesProductDto> findProductsWithParamsByName(@Param("productName") String productName);
}

// PTCLadies DTO (com.yourapp.ptcladies.dto)
public class PTCLadiesProductDto {
    private String productId;
    private String productName;
    private String paramName;
    private String paramValue;
    private String source;
    
    public PTCLadiesProductDto(String productId, String productName, String paramName, String paramValue) {
        this.productId = productId;
        this.productName = productName;
        this.paramName = paramName;
        this.paramValue = paramValue;
    }
    
    // Getters and setters
    public String getProductId() { return productId; }
    public void setProductId(String productId) { this.productId = productId; }
    
    public String getProductName() { return productName; }
    public void setProductName(String productName) { this.productName = productName; }
    
    public String getParamName() { return paramName; }
    public void setParamName(String paramName) { this.paramName = paramName; }
    
    public String getParamValue() { return paramValue; }
    public void setParamValue(String paramValue) { this.paramValue = paramValue; }
    
    public String getSource() { return source; }
    public void setSource(String source) { this.source = source; }
}

// PTCLadies Service (com.yourapp.ptcladies.service)
@Service
@Transactional
public class PTCLadiesService {
    
    @Autowired
    private PTCLadiesWsProductRepository wsProductRepository;
    
    @Autowired
    private DatabaseContextService databaseContextService;
    
    public List<PTCLadiesProductDto> getProductsWithParams(String prodLine, String environment) {
        return databaseContextService.executeWithDatabase(prodLine, environment, "ptcladies", () -> {
            List<PTCLadiesProductDto> results = wsProductRepository.findProductsWithParams();
            results.forEach(dto -> dto.setSource(prodLine + "-" + environment + "-ptcladies"));
            return results;
        });
    }
    
    public List<PTCLadiesProductDto> searchProductsWithParams(String prodLine, String environment, String productName) {
        return databaseContextService.executeWithDatabase(prodLine, environment, "ptcladies", () -> {
            List<PTCLadiesProductDto> results = wsProductRepository.findProductsWithParamsByName(productName);
            results.forEach(dto -> dto.setSource(prodLine + "-" + environment + "-ptcladies"));
            return results;
        });
    }
}

// ====================================================================
// Sampling Package Structure
// ====================================================================

// Sampling Configuration (com.yourapp.sampling.config)
@Configuration
@EnableJpaRepositories(
    basePackages = "com.yourapp.sampling.repository",
    entityManagerFactoryRef = "samplingEntityManagerFactory",
    transactionManagerRef = "samplingTransactionManager"
)
@EntityScan("com.yourapp.sampling.entity")
public class SamplingJpaConfiguration {

    @Bean(name = "samplingEntityManagerFactory")
    public LocalContainerEntityManagerFactoryBean samplingEntityManagerFactory(
            @Qualifier("dataSource") DataSource dataSource) {
        
        LocalContainerEntityManagerFactoryBean em = new LocalContainerEntityManagerFactoryBean();
        em.setDataSource(dataSource);
        em.setPackagesToScan("com.yourapp.sampling.entity");
        em.setPersistenceUnitName("sampling");
        
        HibernateJpaVendorAdapter vendorAdapter = new HibernateJpaVendorAdapter();
        em.setJpaVendorAdapter(vendorAdapter);
        
        Properties props = new Properties();
        props.setProperty("hibernate.hbm2ddl.auto", "none");
        props.setProperty("hibernate.dialect", "org.hibernate.dialect.SQLServerDialect");
        em.setJpaProperties(props);
        
        return em;
    }

    @Bean(name = "samplingTransactionManager")
    public PlatformTransactionManager samplingTransactionManager(
            @Qualifier("samplingEntityManagerFactory") EntityManagerFactory entityManagerFactory) {
        return new JpaTransactionManager(entityManagerFactory);
    }
}

// Sampling Entities (com.yourapp.sampling.entity)
@Entity
@Table(name = "sample_data")
public class SamplingData {
    @Id
    @Column(name = "sample_id")
    private String sampleId;
    
    @Column(name = "sample_type")
    private String sampleType;
    
    @Column(name = "test_result")
    private String testResult;
    
    @Column(name = "sample_date")
    @Temporal(TemporalType.TIMESTAMP)
    private Date sampleDate;
    
    @Column(name = "batch_id")
    private String batchId;
    
    // Getters and setters
    public String getSampleId() { return sampleId; }
    public void setSampleId(String sampleId) { this.sampleId = sampleId; }
    
    public String getSampleType() { return sampleType; }
    public void setSampleType(String sampleType) { this.sampleType = sampleType; }
    
    public String getTestResult() { return testResult; }
    public void setTestResult(String testResult) { this.testResult = testResult; }
    
    public Date getSampleDate() { return sampleDate; }
    public void setSampleDate(Date sampleDate) { this.sampleDate = sampleDate; }
    
    public String getBatchId() { return batchId; }
    public void setBatchId(String batchId) { this.batchId = batchId; }
}

@Entity
@Table(name = "batch_info")
public class SamplingBatchInfo {
    @Id
    @Column(name = "batch_id")
    private String batchId;
    
    @Column(name = "batch_name")
    private String batchName;
    
    @Column(name = "production_date")
    @Temporal(TemporalType.DATE)
    private Date productionDate;
    
    // Getters and setters
    public String getBatchId() { return batchId; }
    public void setBatchId(String batchId) { this.batchId = batchId; }
    
    public String getBatchName() { return batchName; }
    public void setBatchName(String batchName) { this.batchName = batchName; }
    
    public Date getProductionDate() { return productionDate; }
    public void setProductionDate(Date productionDate) { this.productionDate = productionDate; }
}

// Sampling Repository (com.yourapp.sampling.repository)
@Repository
public interface SamplingDataRepository extends JpaRepository<SamplingData, String> {
    
    @Query("SELECT new com.yourapp.sampling.dto.SamplingDto(s.sampleId, s.sampleType, s.testResult, s.sampleDate, b.batchName) " +
           "FROM SamplingData s JOIN SamplingBatchInfo b ON s.batchId = b.batchId")
    List<SamplingDto> findSamplingWithBatchInfo();
    
    @Query("SELECT new com.yourapp.sampling.dto.SamplingDto(s.sampleId, s.sampleType, s.testResult, s.sampleDate, b.batchName) " +
           "FROM SamplingData s JOIN SamplingBatchInfo b ON s.batchId = b.batchId " +
           "WHERE s.sampleType LIKE %:sampleType% OR b.batchName LIKE %:searchTerm%")
    List<SamplingDto> findSamplingWithBatchInfoBySearch(@Param("sampleType") String sampleType, 
                                                        @Param("searchTerm") String searchTerm);
}

// Sampling DTO (com.yourapp.sampling.dto)
public class SamplingDto {
    private String sampleId;
    private String sampleType;
    private String testResult;
    private Date sampleDate;
    private String batchName;
    private String source;
    
    public SamplingDto(String sampleId, String sampleType, String testResult, Date sampleDate, String batchName) {
        this.sampleId = sampleId;
        this.sampleType = sampleType;
        this.testResult = testResult;
        this.sampleDate = sampleDate;
        this.batchName = batchName;
    }
    
    // Getters and setters
    public String getSampleId() { return sampleId; }
    public void setSampleId(String sampleId) { this.sampleId = sampleId; }
    
    public String getSampleType() { return sampleType; }
    public void setSampleType(String sampleType) { this.sampleType = sampleType; }
    
    public String getTestResult() { return testResult; }
    public void setTestResult(String testResult) { this.testResult = testResult; }
    
    public Date getSampleDate() { return sampleDate; }
    public void setSampleDate(Date sampleDate) { this.sampleDate = sampleDate; }
    
    public String getBatchName() { return batchName; }
    public void setBatchName(String batchName) { this.batchName = batchName; }
    
    public String getSource() { return source; }
    public void setSource(String source) { this.source = source; }
}

// Sampling Service (com.yourapp.sampling.service)
@Service
@Transactional
public class SamplingService {
    
    @Autowired
    private SamplingDataRepository samplingDataRepository;
    
    @Autowired
    private DatabaseContextService databaseContextService;
    
    public List<SamplingDto> getAllSamplingData(String prodLine, String environment) {
        return databaseContextService.executeWithDatabase(prodLine, environment, "sampling", () -> {
            List<SamplingDto> results = samplingDataRepository.findSamplingWithBatchInfo();
            results.forEach(dto -> dto.setSource(prodLine + "-" + environment + "-sampling"));
            return results;
        });
    }
    
    public List<SamplingDto> searchSamplingData(String prodLine, String environment, String searchTerm) {
        return databaseContextService.executeWithDatabase(prodLine, environment, "sampling", () -> {
            List<SamplingDto> results = samplingDataRepository.findSamplingWithBatchInfoBySearch(searchTerm, searchTerm);
            results.forEach(dto -> dto.setSource(prodLine + "-" + environment + "-sampling"));
            return results;
        });
    }
}

// ====================================================================
// MPPDB Package Structure
// ====================================================================

// MPPDB Configuration (com.yourapp.mppdb.config)
@Configuration
@EnableJpaRepositories(
    basePackages = "com.yourapp.mppdb.repository",
    entityManagerFactoryRef = "mppdbEntityManagerFactory",
    transactionManagerRef = "mppdbTransactionManager"
)
@EntityScan("com.yourapp.mppdb.entity")
public class MppdbJpaConfiguration {

    @Bean(name = "mppdbEntityManagerFactory")
    public LocalContainerEntityManagerFactoryBean mppdbEntityManagerFactory(
            @Qualifier("dataSource") DataSource dataSource) {
        
        LocalContainerEntityManagerFactoryBean em = new LocalContainerEntityManagerFactoryBean();
        em.setDataSource(dataSource);
        em.setPackagesToScan("com.yourapp.mppdb.entity");
        em.setPersistenceUnitName("mppdb");
        
        HibernateJpaVendorAdapter vendorAdapter = new HibernateJpaVendorAdapter();
        em.setJpaVendorAdapter(vendorAdapter);
        
        Properties props = new Properties();
        props.setProperty("hibernate.hbm2ddl.auto", "none");
        props.setProperty("hibernate.dialect", "org.hibernate.dialect.SQLServerDialect");
        em.setJpaProperties(props);
        
        return em;
    }

    @Bean(name = "mppdbTransactionManager")
    public PlatformTransactionManager mppdbTransactionManager(
            @Qualifier("mppdbEntityManagerFactory") EntityManagerFactory entityManagerFactory) {
        return new JpaTransactionManager(entityManagerFactory);
    }
}

// MPPDB Entities (com.yourapp.mppdb.entity)
@Entity
@Table(name = "process_data")
public class MppdbProcessData {
    @Id
    @Column(name = "process_id")
    private String processId;
    
    @Column(name = "process_name")
    private String processName;
    
    @Column(name = "machine_id")
    private String machineId;
    
    @Column(name = "temperature")
    private Double temperature;
    
    @Column(name = "pressure")
    private Double pressure;
    
    @Column(name = "timestamp")
    @Temporal(TemporalType.TIMESTAMP)
    private Date timestamp;
    
    // Getters and setters
    public String getProcessId() { return processId; }
    public void setProcessId(String processId) { this.processId = processId; }
    
    public String getProcessName() { return processName; }
    public void setProcessName(String processName) { this.processName = processName; }
    
    public String getMachineId() { return machineId; }
    public void setMachineId(String machineId) { this.machineId = machineId; }
    
    public Double getTemperature() { return temperature; }
    public void setTemperature(Double temperature) { this.temperature = temperature; }
    
    public Double getPressure() { return pressure; }
    public void setPressure(Double pressure) { this.pressure = pressure; }
    
    public Date getTimestamp() { return timestamp; }
    public void setTimestamp(Date timestamp) { this.timestamp = timestamp; }
}

// MPPDB Repository (com.yourapp.mppdb.repository)
@Repository
public interface MppdbProcessDataRepository extends JpaRepository<MppdbProcessData, String> {
    
    @Query("SELECT m FROM MppdbProcessData m ORDER BY m.timestamp DESC")
    List<MppdbProcessData> findAllOrderByTimestamp();
    
    @Query("SELECT m FROM MppdbProcessData m WHERE m.processName LIKE %:processName% OR m.machineId LIKE %:machineId% ORDER BY m.timestamp DESC")
    List<MppdbProcessData> findByProcessNameOrMachineId(@Param("processName") String processName, 
                                                       @Param("machineId") String machineId);
}

// MPPDB DTO (com.yourapp.mppdb.dto)
public class MppdbDto {
    private String processId;
    private String processName;
    private String machineId;
    private Double temperature;
    private Double pressure;
    private Date timestamp;
    private String source;
    
    // Constructor and getters/setters
    public MppdbDto(MppdbProcessData processData) {
        this.processId = processData.getProcessId();
        this.processName = processData.getProcessName();
        this.machineId = processData.getMachineId();
        this.temperature = processData.getTemperature();
        this.pressure = processData.getPressure();
        this.timestamp = processData.getTimestamp();
    }
    
    // Getters and setters
    public String getProcessId() { return processId; }
    public void setProcessId(String processId) { this.processId = processId; }
    
    public String getProcessName() { return processName; }
    public void setProcessName(String processName) { this.processName = processName; }
    
    public String getMachineId() { return machineId; }
    public void setMachineId(String machineId) { this.machineId = machineId; }
    
    public Double getTemperature() { return temperature; }
    public void setTemperature(Double temperature) { this.temperature = temperature; }
    
    public Double getPressure() { return pressure; }
    public void setPressure(Double pressure) { this.pressure = pressure; }
    
    public Date getTimestamp() { return timestamp; }
    public void setTimestamp(Date timestamp) { this.timestamp = timestamp; }
    
    public String getSource() { return source; }
    public void setSource(String source) { this.source = source; }
}

// MPPDB Service (com.yourapp.mppdb.service)
@Service
@Transactional
public class MppdbService {
    
    @Autowired
    private MppdbProcessDataRepository processDataRepository;
    
    @Autowired
    private DatabaseContextService databaseContextService;
    
    public List<MppdbDto> getAllMppdbData(String prodLine, String environment) {
        return databaseContextService.executeWithDatabase(prodLine, environment, "mppdb", () -> {
            List<MppdbProcessData> processDataList = processDataRepository.findAllOrderByTimestamp();
            List<MppdbDto> results = processDataList.stream()
                .map(MppdbDto::new)
                .collect(Collectors.toList());
            results.forEach(dto -> dto.setSource(prodLine + "-" + environment + "-mppdb"));
            return results;
        });
    }
    
    public List<MppdbDto> searchMppdbData(String prodLine, String environment, String searchTerm) {
        return databaseContextService.executeWithDatabase(prodLine, environment, "mppdb", () -> {
            List<MppdbProcessData> processDataList = processDataRepository.findByProcessNameOrMachineId(searchTerm, searchTerm);
            List<MppdbDto> results = processDataList.stream()
                .map(MppdbDto::new)
                .collect(Collectors.toList());
            results.forEach(dto -> dto.setSource(prodLine + "-" + environment + "-mppdb"));
            return results;
        });
    }
}

// ====================================================================
// MES Package Structure
// ====================================================================

// MES Configuration (com.yourapp.mes.config)
@Configuration
@EnableJpaRepositories(
    basePackages = "com.yourapp.mes.repository",
    entityManagerFactoryRef = "mesEntityManagerFactory",
    transactionManagerRef = "mesTransactionManager"
)
@EntityScan("com.yourapp.mes.entity")
public class MesJpaConfiguration {

    @Bean(name = "mesEntityManagerFactory")
    public LocalContainerEntityManagerFactoryBean mesEntityManagerFactory(
            @Qualifier("dataSource") DataSource dataSource) {
        
        LocalContainerEntityManagerFactoryBean em = new LocalContainerEntityManagerFactoryBean();
        em.setDataSource(dataSource);
        em.setPackagesToScan("com.yourapp.mes.entity");
        em.setPersistenceUnitName("mes");
        
        HibernateJpaVendorAdapter vendorAdapter = new HibernateJpaVendorAdapter();
        em.setJpaVendorAdapter(vendorAdapter);
        
        Properties props = new Properties();
        props.setProperty("hibernate.hbm2ddl.auto", "none");
        props.setProperty("hibernate.dialect", "org.hibernate.dialect.SQLServerDialect");
        em.setJpaProperties(props);
        
        return em;
    }

    @Bean(name = "mesTransactionManager")
    public PlatformTransactionManager mesTransactionManager(
            @Qualifier("mesEntityManagerFactory") EntityManagerFactory entityManagerFactory) {
        return new JpaTransactionManager(entityManagerFactory);
    }
}

// MES Entities (com.yourapp.mes.entity)
@Entity
@Table(name = "work_order")
public class MesWorkOrder {
    @Id
    @Column(name = "work_order_id")
    private String workOrderId;
    
    @Column(name = "product_code")
    private String productCode;
    
    @Column(name = "quantity")
    private Integer quantity;
    
    @Column(name = "status")
    private String status;
    
    @Column(name = "start_date")
    @Temporal(TemporalType.TIMESTAMP)
    private Date startDate;
    
    @Column(name = "completion_date")
    @Temporal(TemporalType.TIMESTAMP)
    private Date completionDate;
    
    // Getters and setters
    public String getWorkOrderId() { return workOrderId; }
    public void setWorkOrderId(String workOrderId) { this.workOrderId = workOrderId; }
    
    public String getProductCode() { return productCode; }
    public void setProductCode(String productCode) { this.productCode = productCode; }
    
    public Integer getQuantity() { return quantity; }
    public void setQuantity(Integer quantity) { this.quantity = quantity; }
    
    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }
    
    public Date getStartDate() { return startDate; }
    public void setStartDate(Date startDate) { this.startDate = startDate; }
    
    public Date getCompletionDate() { return completionDate; }
    public void setCompletionDate(Date completionDate) { this.completionDate = completionDate; }
}

// MES Repository (com.yourapp.mes.repository)
@Repository
public interface MesWorkOrderRepository extends JpaRepository<MesWorkOrder, String> {
    
    @Query("SELECT m FROM MesWorkOrder m ORDER BY m.startDate DESC")
    List<MesWorkOrder> findAllOrderByStartDate();
    
    @Query("SELECT m FROM MesWorkOrder m WHERE m.productCode LIKE %:productCode% OR m.status LIKE %:status% ORDER BY m.startDate DESC")
    List<MesWorkOrder> findByProductCodeOrStatus(@Param("productCode") String productCode, 
                                                @Param("status") String status);
}

// MES DTO (com.yourapp.mes.dto)
public class MesDto {
    private String workOrderId;
    private String productCode;
    private Integer quantity;
    private String status;
    private Date startDate;
    private Date completionDate;
    private String source;
    
    public MesDto(MesWorkOrder workOrder) {
        this.workOrderId = workOrder.getWorkOrderId();
        this.productCode = workOrder.getProductCode();
        this.quantity = workOrder.getQuantity();
        this.status = workOrder.getStatus();
        this.startDate = workOrder.getStartDate();
        this.completionDate = workOrder.getCompletionDate();
    }
    
    // Getters and setters
    public String getWorkOrderId() { return workOrderId; }
    public void setWorkOrderId(String workOrderId) { this.workOrderId = workOrderId; }
    
    public String getProductCode() { return productCode; }
    public void setProductCode(String productCode) { this.productCode = productCode; }
    
    public Integer getQuantity() { return quantity; }
    public void setQuantity(Integer quantity) { this.quantity = quantity; }
    
    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }
    
    public Date getStartDate() { return startDate; }
    public void setStartDate(Date startDate) { this.startDate = startDate; }
    
    public Date getCompletionDate() { return completionDate; }
    public void setCompletionDate(Date completionDate) { this.completionDate = completionDate; }
    
    public String getSource() { return source; }
    public void setSource(String source) { this.source = source; }
}

// MES Service (com.yourapp.mes.service)
@Service
@Transactional
public class MesService {
    
    @Autowired
    private MesWorkOrderRepository workOrderRepository;
    
    @Autowired
    private DatabaseContextService databaseContextService;
    
    public List<MesDto> getAllMesData(String prodLine, String environment) {
        return databaseContextService.executeWithDatabase(prodLine, environment, "mes", () -> {
            List<MesWorkOrder> workOrders = workOrderRepository.findAllOrderByStartDate();
            List<MesDto> results = workOrders.stream()
                .map(MesDto::new)
                .collect(Collectors.toList());
            results.forEach(dto -> dto.setSource(prodLine + "-" + environment + "-mes"));
            return results;
        });
    }
    
    public List<MesDto> searchMesData(String prodLine, String environment, String searchTerm) {
        return databaseContextService.executeWithDatabase(prodLine, environment, "mes", () -> {
            List<MesWorkOrder> workOrders = workOrderRepository.findByProductCodeOrStatus(searchTerm, searchTerm);
            List<MesDto> results = workOrders.stream()
                .map(MesDto::new)
                .collect(Collectors.toList());
            results.forEach(dto -> dto.setSource(prodLine + "-" + environment + "-mes"));
            return results;
        });
    }
}

// ====================================================================
// Updated Controller to handle all database types
// ====================================================================

@Controller
public class EnhancedMultiDatabaseController {
    
    private static final Logger logger = LoggerFactory.getLogger(EnhancedMultiDatabaseController.class);
    
    @Autowired
    private PTCLadiesService ptcLadiesService;
    
    @Autowired
    private SamplingService samplingService;
    
    @Autowired
    private MppdbService mppdbService;
    
    @Autowired
    private MesService mesService;
    
    @Autowired
    private DatabaseContextService databaseContextService;
    
    @Autowired
    private UniversalExportService exportService;

    @GetMapping("/")
    public String index() {
        return "redirect:/database-interface";
    }
    
    @GetMapping("/database-interface")
    public String showDatabaseInterface(Model model) {
        model.addAttribute("productionLines", databaseContextService.getConfiguredProductionLines());
        model.addAttribute("systemDatabases", Arrays.asList("ptcladies", "sampling", "mppdb", "mes"));
        model.addAttribute("environments", Arrays.asList("test", "prod"));
        
        // Initialize empty data
        model.addAttribute("selectedProdLine", "");
        model.addAttribute("selectedDatabase", "");
        model.addAttribute("searchTerm", "");
        model.addAttribute("testData", new ArrayList<>());
        model.addAttribute("prodData", new ArrayList<>());
        
        return "database-interface";
    }
    
    @PostMapping("/query-database")
    public String queryDatabase(@RequestParam String prodLine,
                               @RequestParam String database,
                               @RequestParam(required = false) String searchTerm,
                               Model model) {
        
        logger.info("Querying database: {}-{}, search term: {}", prodLine, database, searchTerm);
        
        try {
            validateDatabaseAccess(prodLine, database);
            
            List<Object> testData = new ArrayList<>();
            List<Object> prodData = new ArrayList<>();
            
            // Query based on selected system database
            switch (database.toLowerCase()) {
                case "ptcladies":
                    testData = queryPTCLadies(prodLine, "test", searchTerm);
                    prodData = queryPTCLadies(prodLine, "prod", searchTerm);
                    break;
                case "sampling":
                    testData = querySampling(prodLine, "test", searchTerm);
                    prodData = querySampling(prodLine, "prod", searchTerm);
                    break;
                case "mppdb":
                    testData = queryMppdb(prodLine, "test", searchTerm);
                    prodData = queryMppdb(prodLine, "prod", searchTerm);
                    break;
                case "mes":
                    testData = queryMes(prodLine, "test", searchTerm);
                    prodData = queryMes(prodLine, "prod", searchTerm);
                    break;
                default:
                    throw new IllegalArgumentException("Unsupported database: " + database);
            }
            
            // Add data to model
            model.addAttribute("testData", testData);
            model.addAttribute("prodData", prodData);
            model.addAttribute("selectedProdLine", prodLine);
            model.addAttribute("selectedDatabase", database);
            model.addAttribute("searchTerm", searchTerm != null ? searchTerm : "");
            model.addAttribute("successMessage", String.format("Successfully loaded %d test and %d production records from %s", 
                testData.size(), prodData.size(), database.toUpperCase()));
            
        } catch (Exception e) {
            logger.error("Error querying database {}-{}: {}", prodLine, database, e.getMessage(), e);
            model.addAttribute("errorMessage", "Error querying database: " + e.getMessage());
            model.addAttribute("testData", new ArrayList<>());
            model.addAttribute("prodData", new ArrayList<>());
        }
        
        // Re-populate form data
        model.addAttribute("productionLines", databaseContextService.getConfiguredProductionLines());
        model.addAttribute("systemDatabases", Arrays.asList("ptcladies", "sampling", "mppdb", "mes"));
        model.addAttribute("environments", Arrays.asList("test", "prod"));
        
        return "database-interface";
    }
    
    private void validateDatabaseAccess(String prodLine, String database) {
        List<String> configuredDatabases = databaseContextService.getConfiguredDatabases(prodLine, "test");
        if (!configuredDatabases.contains(database)) {
            throw new IllegalArgumentException("Database '" + database + "' is not configured for production line '" + prodLine + "'");
        }
    }
    
    private List<Object> queryPTCLadies(String prodLine, String environment, String searchTerm) {
        if (searchTerm != null && !searchTerm.trim().isEmpty()) {
            return new ArrayList<>(ptcLadiesService.searchProductsWithParams(prodLine, environment, searchTerm));
        } else {
            return new ArrayList<>(ptcLadiesService.getProductsWithParams(prodLine, environment));
        }
    }
    
    private List<Object> querySampling(String prodLine, String environment, String searchTerm) {
        if (searchTerm != null && !searchTerm.trim().isEmpty()) {
            return new ArrayList<>(samplingService.searchSamplingData(prodLine, environment, searchTerm));
        } else {
            return new ArrayList<>(samplingService.getAllSamplingData(prodLine, environment));
        }
    }
    
    private List<Object> queryMppdb(String prodLine, String environment, String searchTerm) {
        if (searchTerm != null && !searchTerm.trim().isEmpty()) {
            return new ArrayList<>(mppdbService.searchMppdbData(prodLine, environment, searchTerm));
        } else {
            return new ArrayList<>(mppdbService.getAllMppdbData(prodLine, environment));
        }
    }
    
    private List<Object> queryMes(String prodLine, String environment, String searchTerm) {
        if (searchTerm != null && !searchTerm.trim().isEmpty()) {
            return new ArrayList<>(mesService.searchMesData(prodLine, environment, searchTerm));
        } else {
            return new ArrayList<>(mesService.getAllMesData(prodLine, environment));
        }
    }
    
    @PostMapping("/export-database")
    @ResponseBody
    public ResponseEntity<byte[]> exportDatabase(@RequestParam String prodLine,
                                               @RequestParam String database,
                                               @RequestParam(required = false) String searchTerm) {
        try {
            logger.info("Exporting database: {}-{}, search term: {}", prodLine, database, searchTerm);
            
            List<Object> testData = new ArrayList<>();
            List<Object> prodData = new ArrayList<>();
            
            // Query based on selected system database
            switch (database.toLowerCase()) {
                case "ptcladies":
                    testData = queryPTCLadies(prodLine, "test", searchTerm);
                    prodData = queryPTCLadies(prodLine, "prod", searchTerm);
                    break;
                case "sampling":
                    testData = querySampling(prodLine, "test", searchTerm);
                    prodData = querySampling(prodLine, "prod", searchTerm);
                    break;
                case "mppdb":
                    testData = queryMppdb(prodLine, "test", searchTerm);
                    prodData = queryMppdb(prodLine, "prod", searchTerm);
                    break;
                case "mes":
                    testData = queryMes(prodLine, "test", searchTerm);
                    prodData = queryMes(prodLine, "prod", searchTerm);
                    break;
                default:
                    throw new IllegalArgumentException("Unsupported database: " + database);
            }
            
            byte[] excelFile = exportService.exportToExcel(testData, prodData, prodLine, database);
            
            String timestamp = LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyyMMdd_HHmmss"));
            String filename = String.format("%s_%s_export_%s.xlsx", prodLine, database, timestamp);
            
            HttpHeaders headers = new HttpHeaders();
            headers.add("Content-Disposition", "attachment; filename=" + filename);
            headers.add("Content-Type", "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet");
            
            return ResponseEntity.ok()
                .headers(headers)
                .body(excelFile);
                
        } catch (Exception e) {
            logger.error("Error exporting database {}-{}: {}", prodLine, database, e.getMessage(), e);
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(("Export failed: " + e.getMessage()).getBytes());
        }
    }
    
    @GetMapping("/api/validate-database")
    @ResponseBody
    public ResponseEntity<Map<String, Object>> validateDatabase(@RequestParam String prodLine,
                                                              @RequestParam String database) {
        Map<String, Object> response = new HashMap<>();
        
        try {
            // Test both environments
            boolean testValid = testDatabaseConnection(prodLine, "test", database);
            boolean prodValid = testDatabaseConnection(prodLine, "prod", database);
            
            response.put("success", true);
            response.put("testEnvironmentValid", testValid);
            response.put("prodEnvironmentValid", prodValid);
            response.put("message", "Database validation completed");
            
            return ResponseEntity.ok(response);
            
        } catch (Exception e) {
            response.put("success", false);
            response.put("error", e.getMessage());
            return ResponseEntity.badRequest().body(response);
        }
    }
    
    private boolean testDatabaseConnection(String prodLine, String environment, String database) {
        try {
            return databaseContextService.executeWithDatabase(prodLine, environment, database, () -> {
                return true;
            });
        } catch (Exception e) {
            logger.warn("Connection test failed for {}-{}-{}: {}", prodLine, environment, database, e.getMessage());
            return false;
        }
    }
}

// ====================================================================
// Package Structure Overview
// ====================================================================

/*
Your project structure should look like this:

src/main/java/
├── com/yourapp/
│   ├── config/
│   │   ├── MultiPackageDatabaseConfiguration.java
│   │   ├── MultiDatabaseProperties.java
│   │   └── DatabaseConfig.java
│   ├── common/
│   │   ├── DatabaseContextHolder.java
│   │   └── DatabaseContextService.java
│   ├── controller/
│   │   └── EnhancedMultiDatabaseController.java
│   ├── ptcladies/
│   │   ├── config/
│   │   │   └── PTCLadiesJpaConfiguration.java
│   │   ├── entity/
│   │   │   ├── PTCLadiesWsProduct.java
│   │   │   └── PTCLadiesParamSetting.java
│   │   ├── repository/
│   │   │   └── PTCLadiesWsProductRepository.java
│   │   ├── dto/
│   │   │   └── PTCLadiesProductDto.java
│   │   └── service/
│   │       └── PTCLadiesService.java
│   ├── sampling/
│   │   ├── config/
│   │   │   └── SamplingJpaConfiguration.java
│   │   ├── entity/
│   │   │   ├── SamplingData.java
│   │   │   └── SamplingBatchInfo.java
│   │   ├── repository/
│   │   │   └── SamplingDataRepository.java
│   │   ├── dto/
│   │   │   └── SamplingDto.java
│   │   └── service/
│   │       └── SamplingService.java
│   ├── mppdb/
│   │   ├── config/
│   │   │   └── MppdbJpaConfiguration.java
│   │   ├── entity/
│   │   │   └── MppdbProcessData.java
│   │   ├── repository/
│   │   │   └── MppdbProcessDataRepository.java
│   │   ├── dto/
│   │   │   └── MppdbDto.java
│   │   └── service/
│   │       └── MppdbService.java
│   ├── mes/
│   │   ├── config/
│   │   │   └── MesJpaConfiguration.java
│   │   ├── entity/
│   │   │   └── MesWorkOrder.java
│   │   ├── repository/
│   │   │   └── MesWorkOrderRepository.java
│   │   ├── dto/
│   │   │   └── MesDto.java
│   │   └── service/
│   │       └── MesService.java
│   └── export/
│       └── UniversalExportService.java

Key Benefits of This Structure:

1. **Complete Separation**: Each system database has its own package with entities, repositories, services
2. **Independent Configurations**: Each database has its own JPA configuration and entity manager
3. **Flexible Entity Mapping**: You can have completely different table structures for each database
4. **Maintainable Code**: Easy to modify one database without affecting others
5. **Scalable Architecture**: Easy to add new databases by creating new packages

Usage Examples:

// PTCLadies specific query
List<PTCLadiesProductDto> products = ptcLadiesService.getProductsWithParams("fl4702", "test");

// Sampling specific query  
List<SamplingDto> samples = samplingService.getAllSamplingData("fl4702", "test");

// MPPDB specific query
List<MppdbDto> processData = mppdbService.getAllMppdbData("fl4702", "test");

// MES specific query
List<MesDto> workOrders = mesService.getAllMesData("fl4702", "test");
*/




// Enhanced Export Service that handles all database types with specific formatting
@Service
public class EnhancedUniversalExportService {
    
    private static final Logger logger = LoggerFactory.getLogger(EnhancedUniversalExportService.class);
    
    public byte[] exportToExcel(List<Object> testData, List<Object> prodData, 
                               String prodLine, String database) throws IOException {
        
        logger.info("Creating Excel export for {}-{} with {} test and {} prod records", 
            prodLine, database, testData.size(), prodData.size());
        
        Workbook workbook = new XSSFWorkbook();
        
        try {
            // Create styles
            CellStyle headerStyle = createHeaderStyle(workbook);
            CellStyle testStyle = createTestStyle(workbook);
            CellStyle prodStyle = createProdStyle(workbook);
            CellStyle dateStyle = createDateStyle(workbook);
            CellStyle numberStyle = createNumberStyle(workbook);
            
            // Create summary sheet
            createSummarySheet(workbook, testData, prodData, prodLine, database);
            
            // Create data sheets based on database type
            switch (database.toLowerCase()) {
                case "ptcladies":
                    createPTCLadiesSheets(workbook, testData, prodData, prodLine, headerStyle, testStyle, prodStyle);
                    break;
                case "sampling":
                    createSamplingSheets(workbook, testData, prodData, prodLine, headerStyle, testStyle, prodStyle, dateStyle);
                    break;
                case "mppdb":
                    createMppdbSheets(workbook, testData, prodData, prodLine, headerStyle, testStyle, prodStyle, dateStyle, numberStyle);
                    break;
                case "mes":
                    createMesSheets(workbook, testData, prodData, prodLine, headerStyle, testStyle, prodStyle, dateStyle, numberStyle);
                    break;
                default:
                    createGenericSheets(workbook, testData, prodData, prodLine, headerStyle, testStyle, prodStyle);
            }
            
            // Create comparison sheet for data analysis
            createComparisonSheet(workbook, testData, prodData, prodLine, database, headerStyle);
            
            ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
            workbook.write(outputStream);
            return outputStream.toByteArray();
            
        } finally {
            workbook.close();
        }
    }
    
    // PTCLadies specific export
    private void createPTCLadiesSheets(Workbook workbook, List<Object> testData, List<Object> prodData,
                                     String prodLine, CellStyle headerStyle, CellStyle testStyle, CellStyle prodStyle) {
        
        List<PTCLadiesProductDto> testProducts = testData.stream()
            .map(obj -> (PTCLadiesProductDto) obj)
            .collect(Collectors.toList());
        List<PTCLadiesProductDto> prodProducts = prodData.stream()
            .map(obj -> (PTCLadiesProductDto) obj)
            .collect(Collectors.toList());
        
        // Test environment sheet
        Sheet testSheet = workbook.createSheet(prodLine + " - PTCLadies Test");
        createPTCLadiesSheet(testSheet, testProducts, headerStyle, testStyle);
        
        // Prod environment sheet
        Sheet prodSheet = workbook.createSheet(prodLine + " - PTCLadies Prod");
        createPTCLadiesSheet(prodSheet, prodProducts, headerStyle, prodStyle);
    }
    
    private void createPTCLadiesSheet(Sheet sheet, List<PTCLadiesProductDto> products, 
                                    CellStyle headerStyle, CellStyle dataStyle) {
        
        // Headers
        Row headerRow = sheet.createRow(0);
        String[] headers = {"Product ID", "Product Name", "Parameter Name", "Parameter Value", "Source"};
        
        for (int i = 0; i < headers.length; i++) {
            Cell cell = headerRow.createCell(i);
            cell.setCellValue(headers[i]);
            cell.setCellStyle(headerStyle);
        }
        
        // Data rows
        int rowNum = 1;
        for (PTCLadiesProductDto product : products) {
            Row row = sheet.createRow(rowNum++);
            createCell(row, 0, product.getProductId(), dataStyle);
            createCell(row, 1, product.getProductName(), dataStyle);
            createCell(row, 2, product.getParamName(), dataStyle);
            createCell(row, 3, product.getParamValue(), dataStyle);
            createCell(row, 4, product.getSource(), dataStyle);
        }
        
        // Auto-size columns
        for (int i = 0; i < headers.length; i++) {
            sheet.autoSizeColumn(i);
        }
    }
    
    // Sampling specific export
    private void createSamplingSheets(Workbook workbook, List<Object> testData, List<Object> prodData,
                                    String prodLine, CellStyle headerStyle, CellStyle testStyle, 
                                    CellStyle prodStyle, CellStyle dateStyle) {
        
        List<SamplingDto> testSamples = testData.stream()
            .map(obj -> (SamplingDto) obj)
            .collect(Collectors.toList());
        List<SamplingDto> prodSamples = prodData.stream()
            .map(obj -> (SamplingDto) obj)
            .collect(Collectors.toList());
        
        // Test environment sheet
        Sheet testSheet = workbook.createSheet(prodLine + " - Sampling Test");
        createSamplingSheet(testSheet, testSamples, headerStyle, testStyle, dateStyle);
        
        // Prod environment sheet
        Sheet prodSheet = workbook.createSheet(prodLine + " - Sampling Prod");
        createSamplingSheet(prodSheet, prodSamples, headerStyle, prodStyle, dateStyle);
    }
    
    private void createSamplingSheet(Sheet sheet, List<SamplingDto> samples, 
                                   CellStyle headerStyle, CellStyle dataStyle, CellStyle dateStyle) {
        
        // Headers
        Row headerRow = sheet.createRow(0);
        String[] headers = {"Sample ID", "Sample Type", "Test Result", "Sample Date", "Batch Name", "Source"};
        
        for (int i = 0; i < headers.length; i++) {
            Cell cell = headerRow.createCell(i);
            cell.setCellValue(headers[i]);
            cell.setCellStyle(headerStyle);
        }
        
        // Data rows
        int rowNum = 1;
        for (SamplingDto sample : samples) {
            Row row = sheet.createRow(rowNum++);
            createCell(row, 0, sample.getSampleId(), dataStyle);
            createCell(row, 1, sample.getSampleType(), dataStyle);
            createCell(row, 2, sample.getTestResult(), dataStyle);
            createDateCell(row, 3, sample.getSampleDate(), dateStyle);
            createCell(row, 4, sample.getBatchName(), dataStyle);
            createCell(row, 5, sample.getSource(), dataStyle);
        }
        
        // Auto-size columns
        for (int i = 0; i < headers.length; i++) {
            sheet.autoSizeColumn(i);
        }
    }
    
    // MPPDB specific export
    private void createMppdbSheets(Workbook workbook, List<Object> testData, List<Object> prodData,
                                 String prodLine, CellStyle headerStyle, CellStyle testStyle, 
                                 CellStyle prodStyle, CellStyle dateStyle, CellStyle numberStyle) {
        
        List<MppdbDto> testProcessData = testData.stream()
            .map(obj -> (MppdbDto) obj)
            .collect(Collectors.toList());
        List<MppdbDto> prodProcessData = prodData.stream()
            .map(obj -> (MppdbDto) obj)
            .collect(Collectors.toList());
        
        // Test environment sheet
        Sheet testSheet = workbook.createSheet(prodLine + " - MPPDB Test");
        createMppdbSheet(testSheet, testProcessData, headerStyle, testStyle, dateStyle, numberStyle);
        
        // Prod environment sheet
        Sheet prodSheet = workbook.createSheet(prodLine + " - MPPDB Prod");
        createMppdbSheet(prodSheet, prodProcessData, headerStyle, prodStyle, dateStyle, numberStyle);
    }
    
    private void createMppdbSheet(Sheet sheet, List<MppdbDto> processDataList, 
                                CellStyle headerStyle, CellStyle dataStyle, CellStyle dateStyle, CellStyle numberStyle) {
        
        // Headers
        Row headerRow = sheet.createRow(0);
        String[] headers = {"Process ID", "Process Name", "Machine ID", "Temperature (°C)", "Pressure (bar)", "Timestamp", "Source"};
        
        for (int i = 0; i < headers.length; i++) {
            Cell cell = headerRow.createCell(i);
            cell.setCellValue(headers[i]);
            cell.setCellStyle(headerStyle);
        }
        
        // Data rows
        int rowNum = 1;
        for (MppdbDto processData : processDataList) {
            Row row = sheet.createRow(rowNum++);
            createCell(row, 0, processData.getProcessId(), dataStyle);
            createCell(row, 1, processData.getProcessName(), dataStyle);
            createCell(row, 2, processData.getMachineId(), dataStyle);
            createNumericCell(row, 3, processData.getTemperature(), numberStyle);
            createNumericCell(row, 4, processData.getPressure(), numberStyle);
            createDateCell(row, 5, processData.getTimestamp(), dateStyle);
            createCell(row, 6, processData.getSource(), dataStyle);
        }
        
        // Auto-size columns
        for (int i = 0; i < headers.length; i++) {
            sheet.autoSizeColumn(i);
        }
    }
    
    // MES specific export
    private void createMesSheets(Workbook workbook, List<Object> testData, List<Object> prodData,
                               String prodLine, CellStyle headerStyle, CellStyle testStyle, 
                               CellStyle prodStyle, CellStyle dateStyle, CellStyle numberStyle) {
        
        List<MesDto> testWorkOrders = testData.stream()
            .map(obj -> (MesDto) obj)
            .collect(Collectors.toList());
        List<MesDto> prodWorkOrders = prodData.stream()
            .map(obj -> (MesDto) obj)
            .collect(Collectors.toList());
        
        // Test environment sheet
        Sheet testSheet = workbook.createSheet(prodLine + " - MES Test");
        createMesSheet(testSheet, testWorkOrders, headerStyle, testStyle, dateStyle, numberStyle);
        
        // Prod environment sheet
        Sheet prodSheet = workbook.createSheet(prodLine + " - MES Prod");
        createMesSheet(prodSheet, prodWorkOrders, headerStyle, prodStyle, dateStyle, numberStyle);
    }
    
    private void createMesSheet(Sheet sheet, List<MesDto> workOrders, 
                              CellStyle headerStyle, CellStyle dataStyle, CellStyle dateStyle, CellStyle numberStyle) {
        
        // Headers
        Row headerRow = sheet.createRow(0);
        String[] headers = {"Work Order ID", "Product Code", "Quantity", "Status", "Start Date", "Completion Date", "Source"};
        
        for (int i = 0; i < headers.length; i++) {
            Cell cell = headerRow.createCell(i);
            cell.setCellValue(headers[i]);
            cell.setCellStyle(headerStyle);
        }
        
        // Data rows
        int rowNum = 1;
        for (MesDto workOrder : workOrders) {
            Row row = sheet.createRow(rowNum++);
            createCell(row, 0, workOrder.getWorkOrderId(), dataStyle);
            createCell(row, 1, workOrder.getProductCode(), dataStyle);
            createNumericCell(row, 2, workOrder.getQuantity() != null ? workOrder.getQuantity().doubleValue() : null, numberStyle);
            createCell(row, 3, workOrder.getStatus(), dataStyle);
            createDateCell(row, 4, workOrder.getStartDate(), dateStyle);
            createDateCell(row, 5, workOrder.getCompletionDate(), dateStyle);
            createCell(row, 6, workOrder.getSource(), dataStyle);
        }
        
        // Auto-size columns
        for (int i = 0; i < headers.length; i++) {
            sheet.autoSizeColumn(i);
        }
    }
    
    // Generic export for unsupported database types
    private void createGenericSheets(Workbook workbook, List<Object> testData, List<Object> prodData,
                                   String prodLine, CellStyle headerStyle, CellStyle testStyle, CellStyle prodStyle) {
        
        Sheet testSheet = workbook.createSheet(prodLine + " - Test Data");
        Sheet prodSheet = workbook.createSheet(prodLine + " - Production Data");
        
        createGenericDataSheet(testSheet, testData, headerStyle, testStyle);
        createGenericDataSheet(prodSheet, prodData, headerStyle, prodStyle);
    }
    
    private void createGenericDataSheet(Sheet sheet, List<Object> data, CellStyle headerStyle, CellStyle dataStyle) {
        if (data.isEmpty()) {
            Row row = sheet.createRow(0);
            createCell(row, 0, "No data available", headerStyle);
            return;
        }
        
        // Simple string representation for generic objects
        Row headerRow = sheet.createRow(0);
        createCell(headerRow, 0, "Data", headerStyle);
        
        int rowNum = 1;
        for (Object item : data) {
            Row row = sheet.createRow(rowNum++);
            createCell(row, 0, item.toString(), dataStyle);
        }
        
        sheet.autoSizeColumn(0);
    }
    
    // Comparison sheet for data analysis
    private void createComparisonSheet(Workbook workbook, List<Object> testData, List<Object> prodData,
                                     String prodLine, String database, CellStyle headerStyle) {
        
        Sheet compSheet = workbook.createSheet("Data Analysis");
        
        // Create analysis based on database type
        switch (database.toLowerCase()) {
            case "ptcladies":
                createPTCLadiesAnalysis(compSheet, testData, prodData, headerStyle);
                break;
            case "sampling":
                createSamplingAnalysis(compSheet, testData, prodData, headerStyle);
                break;
            case "mppdb":
                createMppdbAnalysis(compSheet, testData, prodData, headerStyle);
                break;
            case "mes":
                createMesAnalysis(compSheet, testData, prodData, headerStyle);
                break;
            default:
                createGenericAnalysis(compSheet, testData, prodData, headerStyle);
        }
    }
    
    private void createPTCLadiesAnalysis(Sheet sheet, List<Object> testData, List<Object> prodData, CellStyle headerStyle) {
        List<PTCLadiesProductDto> testProducts = testData.stream().map(obj -> (PTCLadiesProductDto) obj).collect(
