#!/usr/bin/env python3
"""
Simple OSCAR Database Connection Test
Tests Google Cloud SQL PostgreSQL connection with IAM authentication
"""

import sys
import os

def check_and_install_packages():
    """Check and install required packages"""
    required_packages = [
        ('google-cloud-sql-connector', 'google.cloud.sql.connector'),
        ('sqlalchemy', 'sqlalchemy'),
        ('psycopg2-binary', 'psycopg2'),
        ('google-auth', 'google.auth')
    ]
    
    missing_packages = []
    
    for package_name, import_name in required_packages:
        try:
            __import__(import_name)
            print(f"✅ {package_name} - installed")
        except ImportError:
            print(f"❌ {package_name} - missing")
            missing_packages.append(package_name)
    
    if missing_packages:
        print(f"\n📦 Installing missing packages: {', '.join(missing_packages)}")
        for package in missing_packages:
            os.system(f"pip install {package}")
        print("✅ Installation complete!")

# Check packages first
print("🔍 Checking required packages...")
check_and_install_packages()

try:
    from google.cloud.sql.connector import Connector
    from sqlalchemy import create_engine, text
    from google.auth import default
    from google.auth.transport.requests import Request
    import time
    from datetime import datetime
except ImportError as e:
    print(f"❌ Import error: {e}")
    print("Please run: pip install google-cloud-sql-connector sqlalchemy psycopg2-binary google-auth")
    sys.exit(1)

class SimpleOSCARTester:
    def __init__(self):
        """Initialize OSCAR database tester"""
        print("\n🚀 OSCAR Database Connection Tester")
        print("=" * 50)
        
        # OSCAR Database Configuration (from your images)
        self.project = "prj-dv-oscar-0302"
        self.region = "us-central1"  # Update if different
        self.instance = "oscar-0302"
        self.database = "oscar"  # Update with actual database name
        self.user = "sv-1316-gsa-com1-3237@prj-dv-oscar-0302.iam"
        
        self.instance_connection_name = f"{self.project}:{self.region}:{self.instance}"
        self.connector = None
        self.engine = None
        
        print(f"📊 Configuration:")
        print(f"   Project: {self.project}")
        print(f"   Instance: {self.instance_connection_name}")
        print(f"   Database: {self.database}")
        print(f"   User: {self.user}")
    
    def test_authentication(self):
        """Test Google Cloud authentication"""
        print("\n🔐 Testing Google Cloud Authentication...")
        
        try:
            credentials, project = default()
            
            # Refresh credentials if needed
            if hasattr(credentials, 'refresh'):
                credentials.refresh(Request())
            
            print("✅ Authentication successful!")
            print(f"   Detected Project: {project}")
            
            if hasattr(credentials, 'service_account_email'):
                print(f"   Service Account: {credentials.service_account_email}")
            
            return True
            
        except Exception as e:
            print(f"❌ Authentication failed: {e}")
            print("\n🔧 To fix authentication:")
            print("   1. Install Google Cloud SDK")
            print("   2. Run: gcloud auth application-default login")
            print("   3. Or set: export GOOGLE_APPLICATION_CREDENTIALS=/path/to/service-account.json")
            return False
    
    def initialize_connector(self):
        """Initialize Cloud SQL Connector"""
        print("\n🔌 Initializing Cloud SQL Connector...")
        
        try:
            self.connector = Connector()
            print("✅ Connector initialized successfully!")
            return True
        except Exception as e:
            print(f"❌ Connector initialization failed: {e}")
            return False
    
    def test_connection(self):
        """Test database connection"""
        print(f"\n🔗 Testing connection to OSCAR database...")
        
        try:
            # Create connection function
            def getconn():
                print("   📡 Establishing connection...")
                conn = self.connector.connect(
                    self.instance_connection_name,
                    "pg8000",
                    user=self.user,
                    db=self.database,
                    enable_iam_auth=True,
                )
                return conn
            
            # Create SQLAlchemy engine
            print("   🏗️  Creating database engine...")
            self.engine = create_engine(
                "postgresql+pg8000://",
                creator=getconn,
                pool_size=5,
                max_overflow=2,
                pool_pre_ping=True,
                pool_recycle=300
            )
            
            # Test the connection
            print("   🧪 Testing connection...")
            with self.engine.connect() as connection:
                result = connection.execute(text("SELECT 1 as test, current_timestamp as server_time, current_user as db_user"))
                row = result.fetchone()
                
                print("✅ Database connection successful!")
                print(f"   Test Query Result: {row[0]}")
                print(f"   Server Time: {row[1]}")
                print(f"   Connected User: {row[2]}")
                
                return True
                
        except Exception as e:
            print(f"❌ Database connection failed: {e}")
            print(f"   Error Type: {type(e).__name__}")
            
            # Common error solutions
            if "could not connect to server" in str(e).lower():
                print("\n🔧 Possible solutions:")
                print("   1. Check if Cloud SQL instance is running")
                print("   2. Verify region and instance name")
                print("   3. Check firewall rules")
            elif "authentication failed" in str(e).lower():
                print("\n🔧 Possible solutions:")
                print("   1. Verify IAM user has Cloud SQL Client role")
                print("   2. Check if IAM authentication is enabled on instance")
                print("   3. Verify user email format")
            elif "database" in str(e).lower() and "does not exist" in str(e).lower():
                print("\n🔧 Possible solutions:")
                print("   1. Check if database name is correct")
                print("   2. Create the database if it doesn't exist")
            
            return False
    
    def test_oscar_queries(self):
        """Test OSCAR-specific queries"""
        print(f"\n📋 Testing OSCAR-specific queries...")
        
        if not self.engine:
            print("❌ No database connection available")
            return False
        
        try:
            with self.engine.connect() as connection:
                # Test 1: Check database info
                print("   🔍 Testing database information...")
                result = connection.execute(text("""
                    SELECT 
                        current_database() as database_name,
                        current_user as current_user,
                        version() as postgres_version
                """))
                row = result.fetchone()
                print(f"   Database: {row[0]}")
                print(f"   User: {row[1]}")
                print(f"   PostgreSQL Version: {row[2][:50]}...")
                
                # Test 2: Check schemas
                print("   🗂️  Checking available schemas...")
                result = connection.execute(text("""
                    SELECT schema_name 
                    FROM information_schema.schemata 
                    WHERE schema_name NOT IN ('information_schema', 'pg_catalog', 'pg_toast', 'pg_temp_1', 'pg_toast_temp_1')
                    ORDER BY schema_name
                """))
                schemas = [row[0] for row in result.fetchall()]
                print(f"   Available schemas: {', '.join(schemas) if schemas else 'None found'}")
                
                # Test 3: Check for OSCAR tables (if PR01COSRO schema exists)
                if 'PR01COSRO' in [s.upper() for s in schemas] or 'pr01cosro' in schemas:
                    print("   🎯 Found OSCAR schema! Checking tables...")
                    result = connection.execute(text("""
                        SELECT table_name 
                        FROM information_schema.tables 
                        WHERE table_schema = 'PR01COSRO' OR table_schema = 'pr01cosro'
                        ORDER BY table_name
                        LIMIT 10
                    """))
                    tables = [row[0] for row in result.fetchall()]
                    print(f"   OSCAR tables found: {', '.join(tables) if tables else 'None'}")
                    
                    # Test ACTIVE_XML_DATA_STORE if it exists
                    if any('ACTIVE_XML_DATA_STORE' in table.upper() for table in tables):
                        print("   📊 Testing ACTIVE_XML_DATA_STORE table...")
                        try:
                            result = connection.execute(text("""
                                SELECT COUNT(*) as record_count 
                                FROM PR01COSRO.ACTIVE_XML_DATA_STORE 
                                LIMIT 1
                            """))
                            count = result.fetchone()[0]
                            print(f"   Records in ACTIVE_XML_DATA_STORE: {count}")
                        except Exception as e:
                            print(f"   ⚠️  Could not query ACTIVE_XML_DATA_STORE: {e}")
                
                print("✅ OSCAR queries completed successfully!")
                return True
                
        except Exception as e:
            print(f"❌ OSCAR queries failed: {e}")
            return False
    
    def cleanup(self):
        """Clean up connections"""
        print("\n🧹 Cleaning up...")
        
        if self.engine:
            try:
                self.engine.dispose()
                print("✅ Database engine closed")
            except Exception as e:
                print(f"⚠️  Error closing engine: {e}")
        
        if self.connector:
            try:
                self.connector.close()
                print("✅ Connector closed")
            except Exception as e:
                print(f"⚠️  Error closing connector: {e}")
    
    def run_test(self):
        """Run complete test suite"""
        start_time = time.time()
        
        try:
            # Step 1: Test authentication
            if not self.test_authentication():
                return False
            
            # Step 2: Initialize connector
            if not self.initialize_connector():
                return False
            
            # Step 3: Test database connection
            if not self.test_connection():
                return False
            
            # Step 4: Test OSCAR queries
            self.test_oscar_queries()
            
            # Success summary
            end_time = time.time()
            print(f"\n🎉 All tests completed successfully!")
            print(f"⏱️  Total time: {end_time - start_time:.2f} seconds")
            print(f"📅 Test completed at: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
            
            return True
            
        except KeyboardInterrupt:
            print("\n🛑 Test interrupted by user")
            return False
        except Exception as e:
            print(f"\n💥 Unexpected error: {e}")
            return False
        finally:
            self.cleanup()

def main():
    """Main function"""
    tester = SimpleOSCARTester()
    success = tester.run_test()
    
    if success:
        print("\n✅ OSCAR database connectivity test PASSED!")
        print("🚀 Ready to proceed with full application!")
    else:
        print("\n❌ OSCAR database connectivity test FAILED!")
        print("🔧 Please fix the issues above before proceeding.")
    
    return 0 if success else 1

if __name__ == "__main__":
    sys.exit(main())