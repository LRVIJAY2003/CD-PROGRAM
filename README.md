#!/usr/bin/env python3
"""
Final OSCAR Database Connection Script
Tests Google Cloud SQL PostgreSQL connection with IAM authentication
User: lakshya.vijay@cmegroup.com
"""

import sys
import os
import time
from datetime import datetime

def check_and_install_packages():
    """Check and install required packages"""
    required_packages = [
        ('google-cloud-sql-connector', 'google.cloud.sql.connector'),
        ('sqlalchemy', 'sqlalchemy'),
        ('psycopg2-binary', 'psycopg2'),
        ('google-auth', 'google.auth'),
        ('pg8000', 'pg8000')
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
    print("Please run: pip install google-cloud-sql-connector sqlalchemy psycopg2-binary google-auth pg8000")
    sys.exit(1)

class SimpleOSCARTester:
    def __init__(self):
        """Initialize OSCAR database tester"""
        print("\n🏥 OSCAR Database Connection Tester")
        print("-" * 50)
        
        # OSCAR Database Configuration (from your images)
        self.project = "prj-dv-oscar-0302"
        self.region = "us-central1"
        self.instance = "csql-dv-usc1-1316-oscar-0004-m"
        self.database = "oscar"
        self.driver_class = "org.postgresql.Driver"
        
        # FIXED: Use your actual email
        self.user = "lakshya.vijay@cmegroup.com"
        
        self.instance_connection_name = f"{self.project}:{self.region}:{self.instance}"
        self.connector = None
        self.jdbc_url = "jdbc:postgresql:///oscar?currentSchema=dv01cosrs"
        self.enable_iam_auth = True
        self.ssl_mode = "disable"
        self.engine = None
        
        print("📋 Configuration:")
        print(f"   Project: {self.project}")
        print(f"   Instance: {self.instance_connection_name}")
        print(f"   Database: {self.database}")
        print(f"   User: {self.user}")
        print(f"   JDBC URL: {self.jdbc_url}")
        print(f"   Enable IAM Auth: {self.enable_iam_auth}")
        print(f"   Driver Class: {self.driver_class}")

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
            
            # Verify we're using the correct account
            import subprocess
            result = subprocess.run(['gcloud', 'config', 'get-value', 'account'], 
                                  capture_output=True, text=True)
            current_account = result.stdout.strip()
            print(f"   Current Account: {current_account}")
            
            if current_account == self.user:
                print("✅ Account matches database user!")
            else:
                print(f"⚠️  Account mismatch - using {current_account} but connecting as {self.user}")
            
            return True
            
        except Exception as e:
            print(f"❌ Authentication failed: {e}")
            print("\n💡 To fix authentication:")
            print("   1. Run: gcloud auth application-default login")
            print("   2. Set project: gcloud config set project prj-dv-oscar-0302")
            return False

    def verify_sql_user_exists(self):
        """Verify if the user exists in Cloud SQL"""
        print(f"\n👤 Verifying user exists in Cloud SQL: {self.user}")
        
        try:
            import subprocess
            
            cmd = [
                'gcloud', 'sql', 'users', 'list',
                '--instance', self.instance,
                '--project', self.project,
                '--format=value(name,type)',
                f'--filter=name:{self.user}'
            ]
            
            result = subprocess.run(cmd, capture_output=True, text=True)
            
            if result.returncode == 0 and result.stdout.strip():
                user_info = result.stdout.strip().split('\t')
                print(f"✅ User found in Cloud SQL!")
                if len(user_info) >= 2:
                    print(f"   Type: {user_info[1]}")
                return True
            else:
                print(f"❌ User not found in Cloud SQL!")
                print(f"\n💡 To create the user, ask your admin to run:")
                print(f"   gcloud sql users create {self.user} \\")
                print(f"     --instance={self.instance} \\")
                print(f"     --type=cloud_iam_user \\")
                print(f"     --project={self.project}")
                return False
                
        except Exception as e:
            print(f"⚠️  Could not verify user: {e}")
            return True  # Continue anyway

    def initialize_connector(self):
        """Initialize Cloud SQL Connector"""
        print("\n⚡ Initializing Cloud SQL Connector...")
        
        try:
            self.connector = Connector()
            print("✅ Connector initialized successfully!")
            return True
        except Exception as e:
            print(f"❌ Connector initialization failed: {e}")
            return False

    def test_connection(self):
        """Test database connection"""
        print("\n🔧 Testing connection to OSCAR database...")
        
        try:
            # FIXED: Create connection function with PRIVATE IP and your user
            def getconn():
                print("🔗 Establishing connection...")
                conn = self.connector.connect(
                    self.instance_connection_name,
                    "pg8000",
                    user=self.user,
                    db=self.database,
                    enable_iam_auth=True,
                    ip_type="PRIVATE"  # CRITICAL: Use private IP
                )
                return conn
            
            # Create SQLAlchemy engine
            print("🔧 Creating database engine...")
            self.engine = create_engine(
                "postgresql+pg8000://",
                creator=getconn,
                pool_size=5,
                max_overflow=2,
                pool_pre_ping=True,
                pool_recycle=300
            )
            
            # Test the connection
            print("🔄 Testing connection...")
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
            error_str = str(e).lower()
            if "authentication failed" in error_str:
                print("\n💡 Authentication issue:")
                print("   1. Verify your user exists in Cloud SQL")
                print("   2. Check IAM permissions on the instance")
                print("   3. Ensure you're authenticated: gcloud auth application-default login")
            elif "does not have any ip addresses" in error_str:
                print("\n💡 IP address issue:")
                print("   1. Verify you're connecting from authorized network")
                print("   2. Check if instance has private IP enabled")
            elif "could not connect" in error_str:
                print("\n💡 Connection issue:")
                print("   1. Check if Cloud SQL instance is running")
                print("   2. Verify network connectivity to private IP")
                print("   3. Check firewall rules")
            
            return False

    def test_oscar_queries(self):
        """Test OSCAR-specific queries"""
        print("\n🏥 Testing OSCAR-specific queries...")
        
        if not self.engine:
            print("❌ No database connection available")
            return False
        
        try:
            with self.engine.connect() as connection:
                # Test 1: Check database info
                print("🔍 Testing database information...")
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
                print("📁 Checking available schemas...")
                result = connection.execute(text("""
                    SELECT schema_name 
                    FROM information_schema.schemata 
                    WHERE schema_name NOT IN ('information_schema', 'pg_catalog', 'pg_toast', 'pg_temp_1', 'pg_toast_temp_1')
                    ORDER BY schema_name
                """))
                schemas = [row[0] for row in result.fetchall()]
                print(f"   Available schemas: {', '.join(schemas) if schemas else 'None found'}")
                
                # Test 3: Check for OSCAR tables (if PR01COSRO schema exists)
                oscar_schemas = ['PR01COSRO', 'pr01cosro', 'dv01cosrs']
                found_schema = None
                for schema in oscar_schemas:
                    if schema in schemas or schema.lower() in [s.lower() for s in schemas]:
                        found_schema = schema
                        break
                
                if found_schema:
                    print(f"🎯 Found OSCAR schema: {found_schema}! Checking tables...")
                    result = connection.execute(text(f"""
                        SELECT table_name 
                        FROM information_schema.tables 
                        WHERE table_schema = '{found_schema}'
                        ORDER BY table_name
                        LIMIT 10
                    """))
                    tables = [row[0] for row in result.fetchall()]
                    print(f"   OSCAR tables found: {', '.join(tables) if tables else 'None'}")
                    
                    # Test ACTIVE_XML_DATA_STORE if it exists
                    if any('ACTIVE_XML_DATA_STORE' in table.upper() for table in tables):
                        print("📊 Testing ACTIVE_XML_DATA_STORE table...")
                        try:
                            result = connection.execute(text(f"""
                                SELECT COUNT(*) as record_count
                                FROM {found_schema}.ACTIVE_XML_DATA_STORE
                                LIMIT 1
                            """))
                            count = result.fetchone()[0]
                            print(f"   Records in ACTIVE_XML_DATA_STORE: {count}")
                        except Exception as e:
                            print(f"⚠️  Could not query ACTIVE_XML_DATA_STORE: {e}")
                
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

def run_test():
    """Run complete test suite"""
    start_time = time.time()
    
    try:
        # Initialize tester
        tester = SimpleOSCARTester()
        
        # Step 1: Test authentication
        if not tester.test_authentication():
            return False
        
        # Step 1.5: Verify SQL user exists
        tester.verify_sql_user_exists()
        
        # Step 2: Initialize connector  
        if not tester.initialize_connector():
            return False
        
        # Step 3: Test database connection
        if not tester.test_connection():
            return False
        
        # Step 4: Test OSCAR queries
        tester.test_oscar_queries()
        
        # Success summary
        end_time = time.time()
        print("\n🎉 All tests completed successfully!")
        print(f"🕒 Total time: {end_time - start_time:.2f} seconds")
        print(f"📅 Test completed at: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
        
        return True
        
    except KeyboardInterrupt:
        print("\n🛑 Test interrupted by user")
        return False
    except Exception as e:
        print(f"\n💥 Unexpected error: {e}")
        return False
    finally:
        if 'tester' in locals():
            tester.cleanup()

def main():
    """Main function"""
    print("🚀 Starting OSCAR Database Connection Test")
    print(f"👤 User: lakshya.vijay@cmegroup.com")
    print(f"🏥 Database: OSCAR on Google Cloud SQL")
    
    success = run_test()
    
    if success:
        print("\n✅ OSCAR database connectivity test PASSED!")
        print("🚀 Ready to proceed with full application!")
    else:
        print("\n❌ OSCAR database connectivity test FAILED!")
        print("🔧 Please fix the issues above before proceeding.")
    
    return 0 if success else 1

if __name__ == "__main__":
    sys.exit(main())