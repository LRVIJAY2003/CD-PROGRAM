#!/usr/bin/env python3
"""
OSCAR Reconciliation Tool - First Page Implementation
Flask backend with auto-fill functionality
"""

from flask import Flask, render_template, request, jsonify
from flask_cors import CORS
from database_connector import OSCARDatabaseConnector
import logging
from datetime import datetime
import os

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

app = Flask(__name__)
CORS(app)
app.config['SECRET_KEY'] = os.getenv('SECRET_KEY', 'your-secret-key-change-in-production')

# Initialize database connector
db_connector = OSCARDatabaseConnector()

@app.route('/')
def index():
    """Main page"""
    return render_template('index.html')

@app.route('/api/health')
def health_check():
    """Health check endpoint"""
    try:
        # Test database connection
        connection_status = db_connector.test_connection()
        
        return jsonify({
            'status': 'healthy' if connection_status else 'unhealthy',
            'timestamp': datetime.now().isoformat(),
            'database_connected': connection_status
        })
    except Exception as e:
        logger.error(f"Health check failed: {e}")
        return jsonify({
            'status': 'unhealthy',
            'timestamp': datetime.now().isoformat(),
            'error': str(e)
        }), 500

@app.route('/api/lookup', methods=['POST'])
def lookup_data():
    """
    Lookup data based on any input field (GUID, GUS_ID, GFID, Session_ID)
    Auto-fill other fields based on the input
    """
    try:
        data = request.get_json()
        
        # Extract input parameters
        input_guid = data.get('guid', '').strip()
        input_gus_id = data.get('gus_id', '').strip()
        input_gfid = data.get('gfid', '').strip()
        input_session_id = data.get('session_id', '').strip()
        lookup_date = data.get('date', datetime.now().strftime('%Y-%m-%d'))
        
        logger.info(f"Lookup request - GUID: {input_guid}, GUS_ID: {input_gus_id}, GFID: {input_gfid}, Session_ID: {input_session_id}")
        
        # Validate that at least one field is provided
        if not any([input_guid, input_gus_id, input_gfid, input_session_id]):
            return jsonify({
                'success': False,
                'error': 'At least one field (GUID, GUS_ID, GFID, or Session_ID) must be provided'
            }), 400
        
        # Determine which field was provided and perform lookup
        result = None
        
        if input_guid:
            result = lookup_by_guid(input_guid, lookup_date)
        elif input_gus_id:
            result = lookup_by_gus_id(input_gus_id, lookup_date)
        elif input_gfid:
            result = lookup_by_gfid(input_gfid, lookup_date)
        elif input_session_id:
            result = lookup_by_session_id(input_session_id, lookup_date)
        
        if result:
            return jsonify({
                'success': True,
                'data': result,
                'timestamp': datetime.now().isoformat()
            })
        else:
            return jsonify({
                'success': False,
                'error': 'No matching records found',
                'timestamp': datetime.now().isoformat()
            })
            
    except Exception as e:
        logger.error(f"Lookup error: {e}")
        return jsonify({
            'success': False,
            'error': str(e)
        }), 500

def lookup_by_guid(guid, date):
    """Lookup data by GUID - Your exact query"""
    try:
        query = """
        SELECT 
            GUID,
            (xpath('//globexUserSignature/globexUserSignatureInfo/globexFirmID/text()', guses.XML))[1]::text as "GFID",
            (xpath('//globexUserSignature/globexUserSignatureInfo/idNumber/text()', guses.XML))[1]::text as "GUS_ID",
            (xpath('//globexUserSignature/globexUserSignatureInfo/exchange/text()', guses.XML))[1]::text as "Exchange",
            namespace,
            CASE 
                WHEN XML IS NOT NULL THEN 'ACTIVE'
                ELSE 'INACTIVE'
            END as status
        FROM pr01cosrs.ACTIVE_XML_DATA_STORE guses 
        WHERE guses.namespace = 'GlobexUserSignature' 
        AND GUID = :guid
        """
        
        results = db_connector.execute_query(query, {'guid': guid})
        
        if results:
            row = results[0]
            return {
                'guid': row.get('GUID') or row.get('guid'),
                'gfid': row.get('GFID') or row.get('gfid'),
                'gus_id': row.get('GUS_ID') or row.get('gus_id'),
                'exchange': row.get('Exchange') or row.get('exchange'),
                'session_id': determine_session_id(row.get('Exchange') or row.get('exchange')),
                'status': row.get('status'),
                'source': 'GUID_LOOKUP'
            }
        
        return None
        
    except Exception as e:
        logger.error(f"GUID lookup error: {e}")
        raise

def lookup_by_gus_id(gus_id, date):
    """Lookup data by GUS_ID"""
    try:
        query = """
        SELECT 
            GUID,
            (xpath('//globexUserSignature/globexUserSignatureInfo/globexFirmID/text()', guses.XML))[1]::text as "GFID",
            (xpath('//globexUserSignature/globexUserSignatureInfo/idNumber/text()', guses.XML))[1]::text as "GUS_ID",
            (xpath('//globexUserSignature/globexUserSignatureInfo/exchange/text()', guses.XML))[1]::text as "Exchange",
            namespace
        FROM pr01cosrs.ACTIVE_XML_DATA_STORE guses 
        WHERE guses.namespace = 'GlobexUserSignature' 
        AND (xpath('//globexUserSignature/globexUserSignatureInfo/idNumber/text()', guses.XML))[1]::text = :gus_id
        """
        
        results = db_connector.execute_query(query, {'gus_id': gus_id})
        
        if results:
            row = results[0]
            return {
                'guid': row.get('GUID') or row.get('guid'),
                'gfid': row.get('GFID') or row.get('gfid'),
                'gus_id': row.get('GUS_ID') or row.get('gus_id'),
                'exchange': row.get('Exchange') or row.get('exchange'),
                'session_id': determine_session_id(row.get('Exchange') or row.get('exchange')),
                'status': 'ACTIVE',
                'source': 'GUS_ID_LOOKUP'
            }
        
        return None
        
    except Exception as e:
        logger.error(f"GUS_ID lookup error: {e}")
        raise

def lookup_by_gfid(gfid, date):
    """Lookup data by GFID"""
    try:
        query = """
        SELECT 
            GUID,
            (xpath('//globexUserSignature/globexUserSignatureInfo/globexFirmID/text()', guses.XML))[1]::text as "GFID",
            (xpath('//globexUserSignature/globexUserSignatureInfo/idNumber/text()', guses.XML))[1]::text as "GUS_ID",
            (xpath('//globexUserSignature/globexUserSignatureInfo/exchange/text()', guses.XML))[1]::text as "Exchange",
            namespace
        FROM pr01cosrs.ACTIVE_XML_DATA_STORE guses 
        WHERE guses.namespace = 'GlobexUserSignature' 
        AND (xpath('//globexUserSignature/globexUserSignatureInfo/globexFirmID/text()', guses.XML))[1]::text = :gfid
        """
        
        results = db_connector.execute_query(query, {'gfid': gfid})
        
        if results:
            row = results[0]
            return {
                'guid': row.get('GUID') or row.get('guid'),
                'gfid': row.get('GFID') or row.get('gfid'),
                'gus_id': row.get('GUS_ID') or row.get('gus_id'),
                'exchange': row.get('Exchange') or row.get('exchange'),
                'session_id': determine_session_id(row.get('Exchange') or row.get('exchange')),
                'status': 'ACTIVE',
                'source': 'GFID_LOOKUP'
            }
        
        return None
        
    except Exception as e:
        logger.error(f"GFID lookup error: {e}")
        raise

def lookup_by_session_id(session_id, date):
    """Lookup data by Session_ID - This might need different queries"""
    try:
        # First try to find by exchange that matches common session patterns
        exchange_mapping = {
            'MDBLZ': 'CME',
            'FIF': 'EBS',
            'MD': 'CME',
            'OE': 'CME'
        }
        
        exchange = exchange_mapping.get(session_id.upper())
        
        if exchange:
            query = """
            SELECT 
                GUID,
                (xpath('//globexUserSignature/globexUserSignatureInfo/globexFirmID/text()', guses.XML))[1]::text as "GFID",
                (xpath('//globexUserSignature/globexUserSignatureInfo/idNumber/text()', guses.XML))[1]::text as "GUS_ID",
                (xpath('//globexUserSignature/globexUserSignatureInfo/exchange/text()', guses.XML))[1]::text as "Exchange",
                namespace
            FROM pr01cosrs.ACTIVE_XML_DATA_STORE guses 
            WHERE guses.namespace = 'GlobexUserSignature' 
            AND (xpath('//globexUserSignature/globexUserSignatureInfo/exchange/text()', guses.XML))[1]::text = :exchange
            LIMIT 1
            """
            
            results = db_connector.execute_query(query, {'exchange': exchange})
            
            if results:
                row = results[0]
                return {
                    'guid': row.get('GUID') or row.get('guid'),
                    'gfid': row.get('GFID') or row.get('gfid'),
                    'gus_id': row.get('GUS_ID') or row.get('gus_id'),
                    'exchange': row.get('Exchange') or row.get('exchange'),
                    'session_id': session_id,
                    'status': 'ACTIVE',
                    'source': 'SESSION_ID_LOOKUP'
                }
        
        return None
        
    except Exception as e:
        logger.error(f"Session_ID lookup error: {e}")
        raise

def determine_session_id(exchange):
    """Determine session_id based on exchange"""
    if not exchange:
        return None
        
    exchange_to_session = {
        'CME': 'MDBLZ',
        'EBS': 'FIF',
        'BTEC': 'MD',
        'CBOT': 'OE'
    }
    
    return exchange_to_session.get(exchange.upper(), 'MDBLZ')

@app.route('/api/validate', methods=['POST'])
def validate_input():
    """Validate input field format"""
    try:
        data = request.get_json()
        field_type = data.get('type')
        value = data.get('value', '').strip()
        
        validation_rules = {
            'guid': {'length': 12, 'pattern': r'^[A-Z0-9]{12}$'},
            'gus_id': {'length': 5, 'pattern': r'^[A-Z0-9]{5}$'},
            'gfid': {'length': 4, 'pattern': r'^[A-Z0-9]{4}$'},
            'session_id': {'length': None, 'pattern': r'^[A-Z0-9]+$'}
        }
        
        if field_type not in validation_rules:
            return jsonify({'valid': False, 'message': 'Invalid field type'})
        
        rule = validation_rules[field_type]
        
        # Check length if specified
        if rule['length'] and len(value) != rule['length']:
            return jsonify({
                'valid': False,
                'message': f'{field_type.upper()} must be exactly {rule["length"]} characters'
            })
        
        # Check pattern
        import re
        if not re.match(rule['pattern'], value, re.IGNORECASE):
            return jsonify({
                'valid': False,
                'message': f'{field_type.upper()} format is invalid'
            })
        
        return jsonify({'valid': True, 'message': 'Valid format'})
        
    except Exception as e:
        logger.error(f"Validation error: {e}")
        return jsonify({'valid': False, 'message': 'Validation failed'}), 500

if __name__ == '__main__':
    app.run(debug=True, host='0.0.0.0', port=5000)







#!/usr/bin/env python3
"""
OSCAR Database Connector
Using Google Cloud SQL with IAM authentication
Based on your existing test script
"""

import logging
from typing import List, Dict, Any, Optional
from google.cloud.sql.connector import Connector
from sqlalchemy import create_engine, text
from google.auth import default
from google.auth.transport.requests import Request

logger = logging.getLogger(__name__)

class OSCARDatabaseConnector:
    def __init__(self):
        """Initialize OSCAR database connector"""
        
        # OSCAR Database Configuration (from your test script)
        self.project = "prj-dv-oscar-0302"
        self.region = "us-central1"
        self.instance = "csql-dv-usc1-1316-oscar-0004-m"
        self.database = "pgdb"
        self.user = "lakshya.vijay@cmegroup.com"
        
        self.instance_connection_name = f"{self.project}:{self.region}:{self.instance}"
        self.connector = None
        self.engine = None
        self.schema = "pr01cosrs"
        
        logger.info(f"Initializing OSCAR connector for {self.instance_connection_name}")
        
        # Initialize connection
        self._initialize_connection()
    
    def _initialize_connection(self):
        """Initialize database connection"""
        try:
            logger.info("Initializing Cloud SQL Connector...")
            
            # Test authentication
            credentials, project = default()
            if hasattr(credentials, 'refresh'):
                credentials.refresh(Request())
            
            logger.info("Authentication successful")
            
            # Initialize connector
            self.connector = Connector()
            
            # Create connection function
            def getconn():
                logger.debug("Creating database connection...")
                conn = self.connector.connect(
                    self.instance_connection_name,
                    "pg8000",
                    user=self.user,
                    db=self.database,
                    enable_iam_auth=True,
                    ip_type="PRIVATE"
                )
                return conn
            
            # Create SQLAlchemy engine
            self.engine = create_engine(
                "postgresql+pg8000://",
                creator=getconn,
                pool_size=5,
                max_overflow=2,
                pool_pre_ping=True,
                pool_recycle=300
            )
            
            logger.info("Database engine created successfully")
            
        except Exception as e:
            logger.error(f"Database initialization failed: {e}")
            raise
    
    def test_connection(self) -> bool:
        """Test database connection"""
        try:
            if not self.engine:
                logger.error("Database engine not initialized")
                return False
            
            with self.engine.connect() as connection:
                result = connection.execute(text("SELECT 1 as test"))
                row = result.fetchone()
                
                if row and row[0] == 1:
                    logger.info("Database connection test successful")
                    return True
                else:
                    logger.error("Database connection test failed - unexpected result")
                    return False
                    
        except Exception as e:
            logger.error(f"Database connection test failed: {e}")
            return False
    
    def execute_query(self, query: str, params: Optional[Dict[str, Any]] = None) -> List[Dict[str, Any]]:
        """
        Execute a SQL query and return results
        
        Args:
            query: SQL query string
            params: Query parameters
            
        Returns:
            List of dictionaries representing rows
        """
        try:
            if not self.engine:
                raise Exception("Database engine not initialized")
            
            logger.debug(f"Executing query: {query[:100]}...")
            if params:
                logger.debug(f"Parameters: {params}")
            
            with self.engine.connect() as connection:
                if params:
                    result = connection.execute(text(query), params)
                else:
                    result = connection.execute(text(query))
                
                # Convert to list of dictionaries
                columns = result.keys()
                rows = []
                
                for row in result.fetchall():
                    row_dict = {}
                    for i, column in enumerate(columns):
                        row_dict[column] = row[i]
                    rows.append(row_dict)
                
                logger.info(f"Query executed successfully, returned {len(rows)} rows")
                return rows
                
        except Exception as e:
            logger.error(f"Query execution failed: {e}")
            logger.error(f"Query: {query}")
            logger.error(f"Parameters: {params}")
            raise
    
    def get_available_namespaces(self) -> List[Dict[str, Any]]:
        """Get available namespaces in ACTIVE_XML_DATA_STORE"""
        try:
            query = f"""
            SELECT namespace, COUNT(*) as record_count
            FROM {self.schema}.ACTIVE_XML_DATA_STORE 
            GROUP BY namespace
            ORDER BY record_count DESC
            """
            
            return self.execute_query(query)
            
        except Exception as e:
            logger.error(f"Failed to get namespaces: {e}")
            raise
    
    def get_sample_guids(self, namespace: str = 'GlobexUserSignature', limit: int = 5) -> List[Dict[str, Any]]:
        """Get sample GUIDs for testing"""
        try:
            query = f"""
            SELECT GUID, namespace,
                   CASE 
                       WHEN LENGTH(XML::text) > 100 THEN LEFT(XML::text, 100) || '...'
                       ELSE XML::text
                   END as xml_sample
            FROM {self.schema}.ACTIVE_XML_DATA_STORE 
            WHERE namespace = :namespace
            ORDER BY GUID
            LIMIT :limit
            """
            
            return self.execute_query(query, {'namespace': namespace, 'limit': limit})
            
        except Exception as e:
            logger.error(f"Failed to get sample GUIDs: {e}")
            raise
    
    def check_table_structure(self, table_name: str = 'ACTIVE_XML_DATA_STORE') -> List[Dict[str, Any]]:
        """Check table structure"""
        try:
            query = """
            SELECT column_name, data_type, is_nullable, column_default
            FROM information_schema.columns 
            WHERE table_schema = :schema 
            AND table_name = :table_name
            ORDER BY ordinal_position
            """
            
            return self.execute_query(query, {'schema': self.schema, 'table_name': table_name})
            
        except Exception as e:
            logger.error(f"Failed to check table structure: {e}")
            raise
    
    def get_record_count(self, table_name: str = 'ACTIVE_XML_DATA_STORE') -> int:
        """Get total record count"""
        try:
            query = f"SELECT COUNT(*) as count FROM {self.schema}.{table_name}"
            result = self.execute_query(query)
            
            return result[0]['count'] if result else 0
            
        except Exception as e:
            logger.error(f"Failed to get record count: {e}")
            raise
    
    def test_xml_xpath_functionality(self) -> bool:
        """Test if XML xpath functions work"""
        try:
            query = f"""
            SELECT 
                GUID,
                (xpath('//globexUserSignature/globexUserSignatureInfo/globexFirmID/text()', XML))[1]::text as gfid_test
            FROM {self.schema}.ACTIVE_XML_DATA_STORE 
            WHERE namespace = 'GlobexUserSignature'
            AND XML IS NOT NULL
            LIMIT 1
            """
            
            result = self.execute_query(query)
            
            if result and len(result) > 0:
                logger.info("XML xpath functionality test successful")
                return True
            else:
                logger.warning("XML xpath test returned no results")
                return False
                
        except Exception as e:
            logger.error(f"XML xpath functionality test failed: {e}")
            return False
    
    def cleanup(self):
        """Clean up database connections"""
        try:
            if self.engine:
                self.engine.dispose()
                logger.info("Database engine disposed")
            
            if self.connector:
                self.connector.close()
                logger.info("Database connector closed")
                
        except Exception as e:
            logger.warning(f"Cleanup warning: {e}")
    
    def __enter__(self):
        """Context manager entry"""
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        """Context manager exit"""
        self.cleanup()

# Singleton instance for Flask app
_db_connector_instance = None

def get_db_connector() -> OSCARDatabaseConnector:
    """Get singleton database connector instance"""
    global _db_connector_instance
    
    if _db_connector_instance is None:
        _db_connector_instance = OSCARDatabaseConnector()
    
    return _db_connector_instance

# For backwards compatibility
OSCARConnector = OSCARDatabaseConnector






<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>OSCAR Reconciliation Tool - Lookup</title>
    <link href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css" rel="stylesheet">
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&display=swap" rel="stylesheet">
    <link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">
</head>
<body>
    <!-- Loading Overlay -->
    <div id="loading-overlay" class="loading-overlay hidden">
        <div class="loading-spinner">
            <div class="spinner"></div>
            <p>Looking up data...</p>
        </div>
    </div>

    <!-- Header -->
    <header class="header">
        <div class="container">
            <div class="header-content">
                <div class="logo">
                    <i class="fas fa-search"></i>
                    <h1>OSCAR Data Lookup</h1>
                </div>
                <div class="header-actions">
                    <button class="btn-icon" id="health-check-btn" title="Check System Health">
                        <i class="fas fa-heartbeat"></i>
                    </button>
                    <div class="connection-status" id="connection-status">
                        <div class="status-dot"></div>
                        <span>Checking...</span>
                    </div>
                </div>
            </div>
        </div>
    </header>

    <!-- Main Content -->
    <main class="main-content">
        <div class="container">
            <!-- Welcome Section -->
            <section class="welcome-section fade-in">
                <div class="welcome-content">
                    <h2>OSCAR Trading Data Lookup</h2>
                    <p>Enter any field to automatically populate related trading data</p>
                </div>
            </section>

            <!-- Lookup Form -->
            <section class="lookup-section">
                <div class="lookup-card slide-up">
                    <div class="lookup-header">
                        <h3><i class="fas fa-database"></i> Data Lookup & Auto-Fill</h3>
                        <p>Enter any field below and the system will automatically fill related information</p>
                    </div>
                    
                    <form id="lookup-form">
                        <!-- Date Selection -->
                        <div class="form-row">
                            <div class="form-group full-width">
                                <label for="lookup-date">
                                    <i class="fas fa-calendar"></i> Lookup Date
                                </label>
                                <input type="date" id="lookup-date" name="date" required>
                                <div class="input-info">Select the date for data lookup (defaults to today)</div>
                            </div>
                        </div>

                        <!-- Input Fields Grid -->
                        <div class="form-grid">
                            <div class="form-group">
                                <label for="guid">
                                    <i class="fas fa-key"></i> GUID
                                    <span class="field-length">(12 chars)</span>
                                </label>
                                <div class="input-wrapper">
                                    <input 
                                        type="text" 
                                        id="guid" 
                                        name="guid" 
                                        placeholder="e.g., ABCDEFGH1234" 
                                        maxlength="12"
                                        class="auto-lookup-field"
                                        data-field-type="guid"
                                    >
                                    <div class="input-status" id="guid-status"></div>
                                </div>
                                <div class="input-info">Global Unique Identifier</div>
                            </div>
                            
                            <div class="form-group">
                                <label for="gus-id">
                                    <i class="fas fa-user"></i> GUS ID
                                    <span class="field-length">(5 chars)</span>
                                </label>
                                <div class="input-wrapper">
                                    <input 
                                        type="text" 
                                        id="gus-id" 
                                        name="gus_id" 
                                        placeholder="e.g., ABCDE" 
                                        maxlength="5"
                                        class="auto-lookup-field"
                                        data-field-type="gus_id"
                                    >
                                    <div class="input-status" id="gus-id-status"></div>
                                </div>
                                <div class="input-info">Globex User Signature ID</div>
                            </div>
                            
                            <div class="form-group">
                                <label for="gfid">
                                    <i class="fas fa-building"></i> GFID
                                    <span class="field-length">(4 chars)</span>
                                </label>
                                <div class="input-wrapper">
                                    <input 
                                        type="text" 
                                        id="gfid" 
                                        name="gfid" 
                                        placeholder="e.g., ABCD" 
                                        maxlength="4"
                                        class="auto-lookup-field"
                                        data-field-type="gfid"
                                    >
                                    <div class="input-status" id="gfid-status"></div>
                                </div>
                                <div class="input-info">Globex Firm ID</div>
                            </div>
                            
                            <div class="form-group">
                                <label for="session-id">
                                    <i class="fas fa-plug"></i> Session ID
                                </label>
                                <div class="input-wrapper">
                                    <input 
                                        type="text" 
                                        id="session-id" 
                                        name="session_id" 
                                        placeholder="e.g., MDBLZ, FIF" 
                                        class="auto-lookup-field"
                                        data-field-type="session_id"
                                    >
                                    <div class="input-status" id="session-id-status"></div>
                                </div>
                                <div class="input-info">Trading session identifier</div>
                            </div>
                        </div>

                        <!-- Action Buttons -->
                        <div class="form-actions">
                            <button type="button" class="btn-secondary" id="clear-btn">
                                <i class="fas fa-times"></i> Clear All
                            </button>
                            <button type="submit" class="btn-primary" id="lookup-btn">
                                <i class="fas fa-search"></i> Manual Lookup
                            </button>
                        </div>
                    </form>
                </div>
            </section>

            <!-- Results Section -->
            <section class="results-section" id="results-section" style="display: none;">
                <div class="results-header">
                    <h3><i class="fas fa-database"></i> Lookup Results</h3>
                    <div class="results-actions">
                        <button class="btn-secondary" id="copy-results-btn">
                            <i class="fas fa-copy"></i> Copy
                        </button>
                        <button class="btn-secondary" id="next-step-btn">
                            <i class="fas fa-arrow-right"></i> Next Step
                        </button>
                    </div>
                </div>

                <!-- Results Cards -->
                <div class="results-grid" id="results-grid">
                    <div class="result-card">
                        <div class="card-header">
                            <h4><i class="fas fa-key"></i> GUID</h4>
                        </div>
                        <div class="card-content">
                            <div class="card-value" id="result-guid">-</div>
                            <div class="card-source" id="result-guid-source">-</div>
                        </div>
                    </div>

                    <div class="result-card">
                        <div class="card-header">
                            <h4><i class="fas fa-user"></i> GUS ID</h4>
                        </div>
                        <div class="card-content">
                            <div class="card-value" id="result-gus-id">-</div>
                            <div class="card-source" id="result-gus-id-source">-</div>
                        </div>
                    </div>

                    <div class="result-card">
                        <div class="card-header">
                            <h4><i class="fas fa-building"></i> GFID</h4>
                        </div>
                        <div class="card-content">
                            <div class="card-value" id="result-gfid">-</div>
                            <div class="card-source" id="result-gfid-source">-</div>
                        </div>
                    </div>

                    <div class="result-card">
                        <div class="card-header">
                            <h4><i class="fas fa-plug"></i> Session ID</h4>
                        </div>
                        <div class="card-content">
                            <div class="card-value" id="result-session-id">-</div>
                            <div class="card-source" id="result-session-id-source">-</div>
                        </div>
                    </div>

                    <div class="result-card">
                        <div class="card-header">
                            <h4><i class="fas fa-exchange-alt"></i> Exchange</h4>
                        </div>
                        <div class="card-content">
                            <div class="card-value" id="result-exchange">-</div>
                            <div class="card-source" id="result-exchange-source">-</div>
                        </div>
                    </div>

                    <div class="result-card">
                        <div class="card-header">
                            <h4><i class="fas fa-info-circle"></i> Status</h4>
                        </div>
                        <div class="card-content">
                            <div class="card-value" id="result-status">-</div>
                            <div class="card-source" id="result-status-source">-</div>
                        </div>
                    </div>
                </div>

                <!-- Additional Information -->
                <div class="additional-info" id="additional-info" style="display: none;">
                    <h4><i class="fas fa-info"></i> Additional Information</h4>
                    <div class="info-content" id="info-content">
                        <!-- Dynamic content will be populated here -->
                    </div>
                </div>
            </section>
        </div>
    </main>

    <!-- Toast Container -->
    <div class="toast-container" id="toast-container"></div>

    <!-- Footer -->
    <footer class="footer">
        <div class="container">
            <p>&copy; 2024 OSCAR Reconciliation Tool. Auto-fill functionality powered by real-time database lookups.</p>
        </div>
    </footer>

    <script src="{{ url_for('static', filename='script.js') }}"></script>
</body>
</html>




/* OSCAR Reconciliation Tool - First Page Styles */

/* Reset and Base Styles */
* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
}

:root {
    /* Color Palette */
    --primary-color: #1a365d;
    --primary-light: #2c5aa0;
    --primary-dark: #0f2537;
    --secondary-color: #e53e3e;
    --accent-color: #00b4d8;
    --success-color: #38a169;
    --warning-color: #ed8936;
    --error-color: #e53e3e;
    
    /* Neutral Colors */
    --white: #ffffff;
    --gray-50: #f9fafb;
    --gray-100: #f3f4f6;
    --gray-200: #e5e7eb;
    --gray-300: #d1d5db;
    --gray-400: #9ca3af;
    --gray-500: #6b7280;
    --gray-600: #4b5563;
    --gray-700: #374151;
    --gray-800: #1f2937;
    --gray-900: #111827;
    
    /* Gradients */
    --gradient-primary: linear-gradient(135deg, var(--primary-color) 0%, var(--primary-light) 100%);
    --gradient-success: linear-gradient(135deg, var(--success-color) 0%, #48bb78 100%);
    --gradient-warning: linear-gradient(135deg, var(--warning-color) 0%, #f6ad55 100%);
    --gradient-error: linear-gradient(135deg, var(--error-color) 0%, #fc8181 100%);
    
    /* Typography */
    --font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', 'Roboto', sans-serif;
    
    /* Spacing */
    --spacing-xs: 0.25rem;
    --spacing-sm: 0.5rem;
    --spacing-md: 1rem;
    --spacing-lg: 1.5rem;
    --spacing-xl: 2rem;
    --spacing-2xl: 3rem;
    --spacing-3xl: 4rem;
    
    /* Border Radius */
    --radius-sm: 0.375rem;
    --radius-md: 0.5rem;
    --radius-lg: 0.75rem;
    --radius-xl: 1rem;
    
    /* Shadows */
    --shadow-sm: 0 1px 2px 0 rgba(0, 0, 0, 0.05);
    --shadow-md: 0 4px 6px -1px rgba(0, 0, 0, 0.1), 0 2px 4px -1px rgba(0, 0, 0, 0.06);
    --shadow-lg: 0 10px 15px -3px rgba(0, 0, 0, 0.1), 0 4px 6px -2px rgba(0, 0, 0, 0.05);
    --shadow-xl: 0 20px 25px -5px rgba(0, 0, 0, 0.1), 0 10px 10px -5px rgba(0, 0, 0, 0.04);
    
    /* Transitions */
    --transition-fast: 0.15s ease-in-out;
    --transition-normal: 0.3s ease-in-out;
    --transition-slow: 0.5s ease-in-out;
}

/* Base Typography */
body {
    font-family: var(--font-family);
    font-size: 1rem;
    line-height: 1.6;
    color: var(--gray-800);
    background: linear-gradient(135deg, #f0f9ff 0%, #e0f2fe 100%);
    min-height: 100vh;
}

/* Layout */
.container {
    max-width: 1200px;
    margin: 0 auto;
    padding: 0 var(--spacing-lg);
}

/* Header */
.header {
    background: var(--gradient-primary);
    color: var(--white);
    padding: var(--spacing-lg) 0;
    box-shadow: var(--shadow-lg);
    position: sticky;
    top: 0;
    z-index: 100;
}

.header-content {
    display: flex;
    justify-content: space-between;
    align-items: center;
}

.logo {
    display: flex;
    align-items: center;
    gap: var(--spacing-md);
}

.logo i {
    font-size: 1.5rem;
    color: var(--accent-color);
}

.logo h1 {
    font-size: 1.5rem;
    font-weight: 700;
    margin: 0;
}

.header-actions {
    display: flex;
    align-items: center;
    gap: var(--spacing-lg);
}

.btn-icon {
    background: rgba(255, 255, 255, 0.1);
    border: 1px solid rgba(255, 255, 255, 0.2);
    color: var(--white);
    padding: var(--spacing-sm);
    border-radius: var(--radius-md);
    cursor: pointer;
    transition: var(--transition-normal);
    backdrop-filter: blur(10px);
}

.btn-icon:hover {
    background: rgba(255, 255, 255, 0.2);
    transform: translateY(-2px);
}

.connection-status {
    display: flex;
    align-items: center;
    gap: var(--spacing-sm);
    background: rgba(255, 255, 255, 0.1);
    padding: var(--spacing-sm) var(--spacing-md);
    border-radius: var(--radius-lg);
    backdrop-filter: blur(10px);
}

.status-dot {
    width: 8px;
    height: 8px;
    border-radius: 50%;
    background: var(--warning-color);
    animation: pulse 2s infinite;
}

.status-dot.connected {
    background: var(--success-color);
}

.status-dot.disconnected {
    background: var(--error-color);
}

@keyframes pulse {
    0%, 100% { opacity: 1; }
    50% { opacity: 0.5; }
}

/* Main Content */
.main-content {
    padding: var(--spacing-2xl) 0;
    min-height: calc(100vh - 200px);
}

/* Welcome Section */
.welcome-section {
    text-align: center;
    margin-bottom: var(--spacing-2xl);
}

.welcome-content h2 {
    font-size: 1.875rem;
    font-weight: 700;
    color: var(--primary-color);
    margin-bottom: var(--spacing-md);
}

.welcome-content p {
    font-size: 1.125rem;
    color: var(--gray-600);
    max-width: 600px;
    margin: 0 auto;
}

/* Lookup Section */
.lookup-section {
    margin-bottom: var(--spacing-2xl);
}

.lookup-card {
    background: var(--white);
    border-radius: var(--radius-xl);
    padding: var(--spacing-2xl);
    box-shadow: var(--shadow-xl);
    border: 1px solid var(--gray-200);
    max-width: 900px;
    margin: 0 auto;
}

.lookup-header {
    text-align: center;
    margin-bottom: var(--spacing-xl);
}

.lookup-header h3 {
    font-size: 1.25rem;
    font-weight: 600;
    color: var(--primary-color);
    margin-bottom: var(--spacing-sm);
}

.lookup-header h3 i {
    margin-right: var(--spacing-sm);
    color: var(--accent-color);
}

.lookup-header p {
    color: var(--gray-600);
    font-size: 0.875rem;
}

/* Form Styles */
.form-row {
    margin-bottom: var(--spacing-lg);
}

.form-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
    gap: var(--spacing-lg);
    margin-bottom: var(--spacing-xl);
}

.form-group {
    display: flex;
    flex-direction: column;
    gap: var(--spacing-sm);
}

.form-group.full-width {
    max-width: 400px;
    margin: 0 auto;
}

.form-group label {
    font-size: 0.875rem;
    font-weight: 500;
    color: var(--gray-700);
    display: flex;
    align-items: center;
    gap: var(--spacing-xs);
}

.form-group label i {
    color: var(--accent-color);
}

.field-length {
    font-size: 0.75rem;
    color: var(--gray-500);
    font-weight: 400;
}

.input-wrapper {
    position: relative;
}

.form-group input {
    width: 100%;
    padding: var(--spacing-md);
    border: 2px solid var(--gray-300);
    border-radius: var(--radius-lg);
    font-size: 1rem;
    transition: var(--transition-normal);
    background: var(--white);
    font-family: var(--font-family);
    text-transform: uppercase;
}

.form-group input:focus {
    outline: none;
    border-color: var(--primary-color);
    box-shadow: 0 0 0 3px rgba(26, 54, 93, 0.1);
}

.form-group input.valid {
    border-color: var(--success-color);
    box-shadow: 0 0 0 3px rgba(56, 161, 105, 0.1);
}

.form-group input.invalid {
    border-color: var(--error-color);
    box-shadow: 0 0 0 3px rgba(229, 62, 62, 0.1);
}

.form-group input.auto-filled {
    background: linear-gradient(135deg, #f0f9ff 0%, #e0f2fe 100%);
    border-color: var(--accent-color);
    box-shadow: 0 0 0 3px rgba(0, 180, 216, 0.1);
}

.input-status {
    position: absolute;
    right: var(--spacing-md);
    top: 50%;
    transform: translateY(-50%);
    font-size: 0.875rem;
}

.input-status.loading {
    color: var(--warning-color);
}

.input-status.success {
    color: var(--success-color);
}

.input-status.error {
    color: var(--error-color);
}

.input-info {
    font-size: 0.75rem;
    color: var(--gray-500);
}

/* Buttons */
.btn-primary {
    background: var(--gradient-primary);
    color: var(--white);
    border: none;
    padding: var(--spacing-md) var(--spacing-xl);
    border-radius: var(--radius-lg);
    font-size: 1rem;
    font-weight: 500;
    cursor: pointer;
    transition: var(--transition-normal);
    display: flex;
    align-items: center;
    justify-content: center;
    gap: var(--spacing-sm);
    box-shadow: var(--shadow-md);
    white-space: nowrap;
}

.btn-primary:hover {
    transform: translateY(-2px);
    box-shadow: var(--shadow-lg);
}

.btn-primary:disabled {
    opacity: 0.6;
    cursor: not-allowed;
    transform: none;
}

.btn-secondary {
    background: var(--white);
    color: var(--gray-700);
    border: 2px solid var(--gray-300);
    padding: var(--spacing-sm) var(--spacing-lg);
    border-radius: var(--radius-lg);
    font-size: 0.875rem;
    font-weight: 500;
    cursor: pointer;
    transition: var(--transition-normal);
    display: flex;
    align-items: center;
    gap: var(--spacing-sm);
}

.btn-secondary:hover {
    border-color: var(--primary-color);
    color: var(--primary-color);
    transform: translateY(-1px);
}

.form-actions {
    display: flex;
    justify-content: center;
    gap: var(--spacing-md);
    margin-top: var(--spacing-xl);
}

/* Results Section */
.results-section {
    margin-bottom: var(--spacing-2xl);
}

.results-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: var(--spacing-xl);
}

.results-header h3 {
    font-size: 1.25rem;
    font-weight: 600;
    color: var(--primary-color);
}

.results-header h3 i {
    margin-right: var(--spacing-sm);
    color: var(--accent-color);
}

.results-actions {
    display: flex;
    gap: var(--spacing-md);
}

.results-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
    gap: var(--spacing-lg);
    margin-bottom: var(--spacing-xl);
}

.result-card {
    background: var(--white);
    border-radius: var(--radius-xl);
    padding: var(--spacing-lg);
    box-shadow: var(--shadow-lg);
    border: 1px solid var(--gray-200);
    transition: var(--transition-normal);
}

.result-card:hover {
    transform: translateY(-4px);
    box-shadow: var(--shadow-xl);
}

.card-header {
    display: flex;
    align-items: center;
    margin-bottom: var(--spacing-md);
}

.card-header h4 {
    font-size: 1rem;
    font-weight: 600;
    color: var(--gray-800);
    margin: 0;
}

.card-header h4 i {
    margin-right: var(--spacing-sm);
    color: var(--accent-color);
}

.card-content {
    text-align: center;
}

.card-value {
    font-size: 1.5rem;
    font-weight: 700;
    color: var(--primary-color);
    margin-bottom: var(--spacing-xs);
    font-family: 'Courier New', monospace;
    word-break: break-all;
}

.card-source {
    font-size: 0.75rem;
    color: var(--gray-500);
    text-transform: uppercase;
    letter-spacing: 0.05em;
}

/* Additional Information */
.additional-info {
    background: var(--gray-50);
    border-radius: var(--radius-lg);
    padding: var(--spacing-lg);
    border: 1px solid var(--gray-200);
}

.additional-info h4 {
    font-size: 1rem;
    font-weight: 600;
    color: var(--gray-800);
    margin-bottom: var(--spacing-md);
}

.additional-info h4 i {
    margin-right: var(--spacing-sm);
    color: var(--accent-color);
}

.info-content {
    font-size: 0.875rem;
    color: var(--gray-700);
    line-height: 1.6;
}

/* Loading Overlay */
.loading-overlay {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: rgba(255, 255, 255, 0.95);
    display: flex;
    justify-content: center;
    align-items: center;
    z-index: 2000;
    backdrop-filter: blur(10px);
}

.loading-spinner {
    text-align: center;
}

.spinner {
    width: 50px;
    height: 50px;
    border: 4px solid var(--gray-300);
    border-top: 4px solid var(--primary-color);
    border-radius: 50%;
    animation: spin 1s linear infinite;
    margin: 0 auto var(--spacing-lg);
}

@keyframes spin {
    0% { transform: rotate(0deg); }
    100% { transform: rotate(360deg); }
}

.loading-spinner p {
    color: var(--gray-700);
    font-weight: 500;
}

/* Toast Notifications */
.toast-container {
    position: fixed;
    top: 20px;
    right: 20px;
    z-index: 3000;
    display: flex;
    flex-direction: column;
    gap: var(--spacing-md);
}

.toast {
    background: var(--white);
    padding: var(--spacing-lg);
    border-radius: var(--radius-lg);
    box-shadow: var(--shadow-xl);
    border-left: 4px solid;
    max-width: 400px;
    animation: slideInRight 0.3s ease-out;
}

.toast.success {
    border-left-color: var(--success-color);
}

.toast.error {
    border-left-color: var(--error-color);
}

.toast.warning {
    border-left-color: var(--warning-color);
}

.toast.info {
    border-left-color: var(--accent-color);
}

/* Animations */
@keyframes fadeIn {
    from { opacity: 0; }
    to { opacity: 1; }
}

@keyframes fadeInUp {
    from {
        opacity: 0;
        transform: translateY(30px);
    }
    to {
        opacity: 1;
        transform: translateY(0);
    }
}

@keyframes slideInRight {
    from { transform: translateX(100%); opacity: 0; }
    to { transform: translateX(0); opacity: 1; }
}

.fade-in {
    animation: fadeIn 0.8s ease-out;
}

.slide-up {
    animation: fadeInUp 0.8s ease-out;
}

/* Utilities */
.hidden {
    display: none !important;
}

/* Footer */
.footer {
    background: var(--gray-800);
    color: var(--white);
    text-align: center;
    padding: var(--spacing-xl) 0;
    margin-top: auto;
}

/* Responsive Design */
@media (max-width: 768px) {
    .container {
        padding: 0 var(--spacing-md);
    }
    
    .header-content {
        flex-direction: column;
        gap: var(--spacing-md);
    }
    
    .form-grid {
        grid-template-columns: 1fr;
    }
    
    .results-header {
        flex-direction: column;
        gap: var(--spacing-md);
        align-items: stretch;
    }
    
    .results-actions {
        justify-content: center;
    }
    
    .form-actions {
        flex-direction: column;
    }
    
    .results-grid {
        grid-template-columns: 1fr;
    }
}

@media (max-width: 480px) {
    .welcome-content h2 {
        font-size: 1.5rem;
    }
    
    .lookup-card {
        padding: var(--spacing-lg);
    }
}






// OSCAR Reconciliation Tool - Auto-Fill Functionality

class OSCARLookupTool {
    constructor() {
        this.currentData = null;
        this.debounceTimers = {};
        this.isAutoFilling = false;
        this.init();
    }

    init() {
        this.bindEvents();
        this.initializeDefaults();
        this.checkSystemHealth();
    }

    initializeDefaults() {
        // Set today's date as default
        const dateInput = document.getElementById('lookup-date');
        const today = new Date().toISOString().split('T')[0];
        dateInput.value = today;
    }

    bindEvents() {
        // Auto-lookup fields
        const autoLookupFields = document.querySelectorAll('.auto-lookup-field');
        autoLookupFields.forEach(field => {
            // Input event for real-time typing
            field.addEventListener('input', (e) => {
                this.handleFieldInput(e.target);
            });

            // Blur event for validation
            field.addEventListener('blur', (e) => {
                this.validateField(e.target);
            });

            // Focus event to clear auto-fill styling
            field.addEventListener('focus', (e) => {
                if (!this.isAutoFilling) {
                    e.target.classList.remove('auto-filled');
                }
            });
        });

        // Manual lookup form
        const form = document.getElementById('lookup-form');
        if (form) {
            form.addEventListener('submit', this.handleManualLookup.bind(this));
        }

        // Clear button
        const clearBtn = document.getElementById('clear-btn');
        if (clearBtn) {
            clearBtn.addEventListener('click', this.clearAllFields.bind(this));
        }

        // Health check button
        const healthBtn = document.getElementById('health-check-btn');
        if (healthBtn) {
            healthBtn.addEventListener('click', this.checkSystemHealth.bind(this));
        }

        // Copy results button
        const copyBtn = document.getElementById('copy-results-btn');
        if (copyBtn) {
            copyBtn.addEventListener('click', this.copyResults.bind(this));
        }

        // Next step button
        const nextBtn = document.getElementById('next-step-btn');
        if (nextBtn) {
            nextBtn.addEventListener('click', this.proceedToNextStep.bind(this));
        }
    }

    handleFieldInput(field) {
        const fieldType = field.dataset.fieldType;
        const value = field.value.trim().toUpperCase();
        
        // Clear previous timer for this field
        if (this.debounceTimers[fieldType]) {
            clearTimeout(this.debounceTimers[fieldType]);
        }

        // Update field value to uppercase
        field.value = value;

        // Real-time validation
        this.validateFieldRealTime(field);

        // Set debounced auto-lookup (wait 800ms after user stops typing)
        this.debounceTimers[fieldType] = setTimeout(() => {
            if (value && this.isValidFieldValue(fieldType, value)) {
                this.performAutoLookup(fieldType, value);
            }
        }, 800);
    }

    validateFieldRealTime(field) {
        const fieldType = field.dataset.fieldType;
        const value = field.value.trim();
        const statusElement = document.getElementById(`${field.id}-status`);

        if (!value) {
            field.className = 'auto-lookup-field';
            statusElement.innerHTML = '';
            return;
        }

        if (this.isValidFieldValue(fieldType, value)) {
            field.classList.add('valid');
            field.classList.remove('invalid');
            statusElement.innerHTML = '<i class="fas fa-check"></i>';
            statusElement.className = 'input-status success';
        } else {
            field.classList.add('invalid');
            field.classList.remove('valid');
            statusElement.innerHTML = '<i class="fas fa-times"></i>';
            statusElement.className = 'input-status error';
        }
    }

    async validateField(field) {
        const fieldType = field.dataset.fieldType;
        const value = field.value.trim();

        if (!value) return;

        try {
            const response = await fetch('/api/validate', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                },
                body: JSON.stringify({
                    type: fieldType,
                    value: value
                })
            });

            const result = await response.json();
            
            if (!result.valid) {
                this.showToast(result.message, 'warning');
            }

        } catch (error) {
            console.error('Validation error:', error);
        }
    }

    isValidFieldValue(fieldType, value) {
        const validationRules = {
            guid: { length: 12, pattern: /^[A-Z0-9]{12}$/ },
            gus_id: { length: 5, pattern: /^[A-Z0-9]{5}$/ },
            gfid: { length: 4, pattern: /^[A-Z0-9]{4}$/ },
            session_id: { length: null, pattern: /^[A-Z0-9]+$/ }
        };

        const rule = validationRules[fieldType];
        if (!rule) return false;

        // Check length if specified
        if (rule.length && value.length !== rule.length) {
            return false;
        }

        // Check pattern
        return rule.pattern.test(value);
    }

    async performAutoLookup(fieldType, value) {
        console.log(`Auto-lookup triggered for ${fieldType}: ${value}`);

        // Show loading state
        this.setFieldStatus(fieldType, 'loading', '<i class="fas fa-spinner fa-spin"></i>');
        
        try {
            const lookupData = {
                date: document.getElementById('lookup-date').value,
                [fieldType]: value
            };

            const response = await fetch('/api/lookup', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                },
                body: JSON.stringify(lookupData)
            });

            const result = await response.json();

            if (result.success && result.data) {
                this.fillRelatedFields(result.data, fieldType);
                this.setFieldStatus(fieldType, 'success', '<i class="fas fa-check"></i>');
                this.showToast(`Auto-filled from ${fieldType.toUpperCase()}`, 'success');
            } else {
                this.setFieldStatus(fieldType, 'error', '<i class="fas fa-times"></i>');
                this.showToast(result.error || 'No matching data found', 'warning');
            }

        } catch (error) {
            console.error('Auto-lookup error:', error);
            this.setFieldStatus(fieldType, 'error', '<i class="fas fa-times"></i>');
            this.showToast('Lookup failed. Please try again.', 'error');
        }
    }

    fillRelatedFields(data, sourceField) {
        this.isAutoFilling = true;
        
        const fieldMappings = {
            'guid': 'guid',
            'gus_id': 'gus-id',
            'gfid': 'gfid',
            'session_id': 'session-id'
        };

        // Fill all fields except the source field
        Object.keys(fieldMappings).forEach(dataKey => {
            const fieldId = fieldMappings[dataKey];
            const field = document.getElementById(fieldId);
            
            if (field && dataKey !== sourceField && data[dataKey]) {
                field.value = data[dataKey];
                field.classList.add('auto-filled');
                
                // Set success status for auto-filled fields
                const statusElement = document.getElementById(`${fieldId}-status`);
                if (statusElement) {
                    statusElement.innerHTML = '<i class="fas fa-magic"></i>';
                    statusElement.className = 'input-status success';
                }
            }
        });

        // Store the current data
        this.currentData = data;

        // Show results
        this.displayResults(data);

        this.isAutoFilling = false;
    }

    setFieldStatus(fieldType, status, icon) {
        const fieldMappings = {
            'guid': 'guid',
            'gus_id': 'gus-id',
            'gfid': 'gfid',
            'session_id': 'session-id'
        };

        const fieldId = fieldMappings[fieldType];
        const statusElement = document.getElementById(`${fieldId}-status`);
        
        if (statusElement) {
            statusElement.innerHTML = icon;
            statusElement.className = `input-status ${status}`;
        }
    }

    async handleManualLookup(e) {
        e.preventDefault();
        
        const formData = new FormData(e.target);
        const lookupData = {
            date: formData.get('date'),
            guid: formData.get('guid')?.trim(),
            gus_id: formData.get('gus_id')?.trim(),
            gfid: formData.get('gfid')?.trim(),
            session_id: formData.get('session_id')?.trim()
        };

        // Check if at least one field is filled
        const hasInput = Object.values(lookupData).some(value => value && value !== lookupData.date);
        if (!hasInput) {
            this.showToast('Please enter at least one field to lookup', 'warning');
            return;
        }

        this.showLoading(true);

        try {
            const response = await fetch('/api/lookup', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                },
                body: JSON.stringify(lookupData)
            });

            const result = await response.json();

            if (result.success && result.data) {
                this.currentData = result.data;
                this.displayResults(result.data);
                this.showToast('Lookup completed successfully', 'success');
            } else {
                this.showToast(result.error || 'No matching data found', 'warning');
            }

        } catch (error) {
            console.error('Manual lookup error:', error);
            this.showToast('Lookup failed. Please try again.', 'error');
        } finally {
            this.showLoading(false);
        }
    }

    displayResults(data) {
        const resultsSection = document.getElementById('results-section');
        
        // Show results section
        resultsSection.style.display = 'block';
        resultsSection.scrollIntoView({ behavior: 'smooth' });

        // Fill result cards
        const resultMappings = {
            'guid': 'result-guid',
            'gus_id': 'result-gus-id',
            'gfid': 'result-gfid',
            'session_id': 'result-session-id',
            'exchange': 'result-exchange',
            'status': 'result-status'
        };

        Object.keys(resultMappings).forEach(dataKey => {
            const elementId = resultMappings[dataKey];
            const valueElement = document.getElementById(elementId);
            const sourceElement = document.getElementById(`${elementId}-source`);
            
            if (valueElement) {
                valueElement.textContent = data[dataKey] || '-';
            }
            
            if (sourceElement) {
                sourceElement.textContent = data.source || 'Database';
            }
        });

        // Show additional info if available
        if (data.additional_info) {
            const additionalInfo = document.getElementById('additional-info');
            const infoContent = document.getElementById('info-content');
            
            additionalInfo.style.display = 'block';
            infoContent.innerHTML = this.formatAdditionalInfo(data.additional_info);
        }

        // Animate result cards
        this.animateResultCards();
    }

    formatAdditionalInfo(info) {
        // Format additional information for display
        if (typeof info === 'object') {
            return Object.keys(info).map(key => 
                `<div><strong>${key}:</strong> ${info[key]}</div>`
            ).join('');
        }
        return String(info);
    }

    animateResultCards() {
        const cards = document.querySelectorAll('.result-card');
        cards.forEach((card, index) => {
            card.style.opacity = '0';
            card.style.transform = 'translateY(20px)';
            
            setTimeout(() => {
                card.style.transition = 'opacity 0.6s ease-out, transform 0.6s ease-out';
                card.style.opacity = '1';
                card.style.transform = 'translateY(0)';
            }, index * 100);
        });
    }

    clearAllFields() {
        // Clear input fields
        const fields = document.querySelectorAll('.auto-lookup-field');
        fields.forEach(field => {
            field.value = '';
            field.className = 'auto-lookup-field';
        });

        // Clear status indicators
        const statusElements = document.querySelectorAll('.input-status');
        statusElements.forEach(element => {
            element.innerHTML = '';
            element.className = 'input-status';
        });

        // Hide results
        const resultsSection = document.getElementById('results-section');
        resultsSection.style.display = 'none';

        // Clear current data
        this.currentData = null;

        // Reset date to today
        const dateInput = document.getElementById('lookup-date');
        const today = new Date().toISOString().split('T')[0];
        dateInput.value = today;

        this.showToast('All fields cleared', 'info');
    }

    async checkSystemHealth() {
        const statusElement = document.querySelector('.connection-status span');
        const statusDot = document.querySelector('.status-dot');

        if (statusElement) {
            statusElement.textContent = 'Checking...';
        }

        if (statusDot) {
            statusDot.className = 'status-dot';
        }

        try {
            const response = await fetch('/api/health');
            const data = await response.json();

            if (response.ok && data.status === 'healthy') {
                if (statusElement) {
                    statusElement.textContent = 'Connected';
                }
                if (statusDot) {
                    statusDot.classList.add('connected');
                }
                
                this.showToast('System health check passed', 'success');
            } else {
                throw new Error('System unhealthy');
            }

        } catch (error) {
            console.error('Health check error:', error);
            
            if (statusElement) {
                statusElement.textContent = 'Disconnected';
            }
            if (statusDot) {
                statusDot.classList.add('disconnected');
            }
            
            this.showToast('System health check failed', 'error');
        }
    }

    copyResults() {
        if (!this.currentData) {
            this.showToast('No data to copy', 'warning');
            return;
        }

        const copyText = Object.keys(this.currentData)
            .map(key => `${key.toUpperCase()}: ${this.currentData[key] || '-'}`)
            .join('\n');

        navigator.clipboard.writeText(copyText).then(() => {
            this.showToast('Results copied to clipboard', 'success');
        }).catch(() => {
            this.showToast('Failed to copy results', 'error');
        });
    }

    proceedToNextStep() {
        if (!this.currentData) {
            this.showToast('No data available for next step', 'warning');
            return;
        }

        // This would navigate to the next page/step
        // For now, just show a message
        this.showToast('Next step functionality coming soon', 'info');
        
        // In the future, this might redirect to comparison tables:
        // window.location.href = '/compare?' + new URLSearchParams(this.currentData);
    }

    showLoading(show, message = 'Looking up data...') {
        const overlay = document.getElementById('loading-overlay');
        const text = overlay?.querySelector('p');

        if (!overlay) return;

        if (show) {
            if (text) {
                text.textContent = message;
            }
            overlay.classList.remove('hidden');
        } else {
            overlay.classList.add('hidden');
        }
    }

    showToast(message, type = 'info', duration = 4000) {
        const container = document.getElementById('toast-container');
        if (!container) return;

        const toast = document.createElement('div');
        toast.className = `toast ${type}`;
        
        const icon = this.getToastIcon(type);
        toast.innerHTML = `
            <div style="display: flex; align-items: center; gap: 0.5rem;">
                <i class="${icon}"></i>
                <span>${message}</span>
            </div>
        `;

        container.appendChild(toast);

        // Auto remove toast
        setTimeout(() => {
            if (container.contains(toast)) {
                toast.style.animation = 'slideOutRight 0.3s ease-out';
                setTimeout(() => {
                    if (container.contains(toast)) {
                        container.removeChild(toast);
                    }
                }, 300);
            }
        }, duration);

        // Click to dismiss
        toast.addEventListener('click', () => {
            if (container.contains(toast)) {
                container.removeChild(toast);
            }
        });
    }

    getToastIcon(type) {
        const icons = {
            success: 'fas fa-check-circle',
            error: 'fas fa-times-circle',
            warning: 'fas fa-exclamation-triangle',
            info: 'fas fa-info-circle'
        };
        
        return icons[type] || icons.info;
    }

    // Utility method to debounce function calls
    debounce(func, wait) {
        let timeout;
        return function executedFunction(...args) {
            const later = () => {
                clearTimeout(timeout);
                func(...args);
            };
            clearTimeout(timeout);
            timeout = setTimeout(later, wait);
        };
    }
}

// CSS for toast slide-out animation
const style = document.createElement('style');
style.textContent = `
    @keyframes slideOutRight {
        from { transform: translateX(0); opacity: 1; }
        to { transform: translateX(100%); opacity: 0; }
    }
`;
document.head.appendChild(style);

// Initialize the application when DOM is loaded
document.addEventListener('DOMContentLoaded', () => {
    // Add page load animation
    document.body.style.opacity = '0';
    document.body.style.transition = 'opacity 0.3s ease-in';
    
    setTimeout(() => {
        document.body.style.opacity = '1';
    }, 100);

    // Initialize the main application
    window.oscarLookupTool = new OSCARLookupTool();

    console.log('OSCAR Lookup Tool initialized successfully');
});









# OSCAR Reconciliation Tool - First Page Dependencies

# Core Flask Framework
Flask==2.3.3
Flask-CORS==4.0.0

# Google Cloud SQL Connector
google-cloud-sql-connector==1.4.3
sqlalchemy==2.0.21
pg8000==1.30.2

# Google Cloud Authentication
google-auth==2.23.4
google-auth-oauthlib==1.1.0

# Additional utilities
python-dateutil==2.8.2

# Development and Testing (optional)
python-dotenv==1.0.0