#!/usr/bin/env python3
"""
Enhanced OSCAR Database Connection Script
Tests Google Cloud SQL PostgreSQL connection with IAM authentication
Includes specific XML xpath query testing
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

class EnhancedOSCARTester:
    def __init__(self):
        """Initialize OSCAR database tester"""
        print("\n🏥 Enhanced OSCAR Database Connection Tester")
        print("-" * 60)
        
        # OSCAR Database Configuration (from your images)
        self.project = "prj-dv-oscar-0302"
        self.region = "us-central1"
        self.instance = "csql-dv-usc1-1316-oscar-0004-m"
        self.database = "pgdb"
        self.driver_class = "org.postgresql.Driver"
        
        # FIXED: Use your actual email
        self.user = "lakshya.vijay@cmegroup.com"
        
        self.instance_connection_name = f"{self.project}:{self.region}:{self.instance}"
        self.connector = None
        self.jdbc_url = "jdbc:postgresql:///oscar?currentSchema=dv01cosrs"
        self.enable_iam_auth = True
        self.ssl_mode = "disable"
        self.engine = None
        
        # Schema to test (from your query)
        self.oscar_schema = "pr01cosrs"
        
        print("📋 Configuration:")
        print(f"   Project: {self.project}")
        print(f"   Instance: {self.instance_connection_name}")
        print(f"   Database: {self.database}")
        print(f"   Schema: {self.oscar_schema}")
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

    def test_oscar_schema_and_tables(self):
        """Test OSCAR schema and tables availability"""
        print("\n📁 Testing OSCAR schema and tables...")
        
        if not self.engine:
            print("❌ No database connection available")
            return False
        
        try:
            with self.engine.connect() as connection:
                # Test 1: Check if schema exists
                print(f"🔍 Checking if schema '{self.oscar_schema}' exists...")
                result = connection.execute(text("""
                    SELECT schema_name 
                    FROM information_schema.schemata 
                    WHERE schema_name = :schema
                """), {"schema": self.oscar_schema})
                
                schema_exists = result.fetchone() is not None
                
                if schema_exists:
                    print(f"✅ Schema '{self.oscar_schema}' found!")
                else:
                    print(f"❌ Schema '{self.oscar_schema}' not found!")
                    # Check available schemas
                    result = connection.execute(text("""
                        SELECT schema_name 
                        FROM information_schema.schemata 
                        WHERE schema_name NOT IN ('information_schema', 'pg_catalog', 'pg_toast')
                        ORDER BY schema_name
                    """))
                    schemas = [row[0] for row in result.fetchall()]
                    print(f"   Available schemas: {', '.join(schemas) if schemas else 'None found'}")
                    return False
                
                # Test 2: Check for ACTIVE_XML_DATA_STORE table
                print(f"🔍 Checking for ACTIVE_XML_DATA_STORE table...")
                result = connection.execute(text("""
                    SELECT table_name 
                    FROM information_schema.tables 
                    WHERE table_schema = :schema 
                    AND table_name = 'ACTIVE_XML_DATA_STORE'
                """), {"schema": self.oscar_schema})
                
                table_exists = result.fetchone() is not None
                
                if table_exists:
                    print("✅ ACTIVE_XML_DATA_STORE table found!")
                else:
                    print("❌ ACTIVE_XML_DATA_STORE table not found!")
                    # Show available tables
                    result = connection.execute(text("""
                        SELECT table_name 
                        FROM information_schema.tables 
                        WHERE table_schema = :schema
                        ORDER BY table_name
                        LIMIT 10
                    """), {"schema": self.oscar_schema})
                    tables = [row[0] for row in result.fetchall()]
                    print(f"   Available tables in {self.oscar_schema}: {', '.join(tables) if tables else 'None'}")
                    return False
                
                # Test 3: Check table structure
                print("🔍 Checking table structure...")
                result = connection.execute(text("""
                    SELECT column_name, data_type, is_nullable
                    FROM information_schema.columns 
                    WHERE table_schema = :schema 
                    AND table_name = 'ACTIVE_XML_DATA_STORE'
                    ORDER BY ordinal_position
                """), {"schema": self.oscar_schema})
                
                columns = result.fetchall()
                print(f"   Table columns ({len(columns)} found):")
                for col in columns[:10]:  # Show first 10 columns
                    print(f"     - {col[0]} ({col[1]}) {'NULL' if col[2] == 'YES' else 'NOT NULL'}")
                
                # Test 4: Check record count
                print("🔍 Checking record count...")
                result = connection.execute(text(f"""
                    SELECT COUNT(*) as total_records
                    FROM {self.oscar_schema}.ACTIVE_XML_DATA_STORE
                """))
                
                count = result.fetchone()[0]
                print(f"   Total records in ACTIVE_XML_DATA_STORE: {count:,}")
                
                return True
                
        except Exception as e:
            print(f"❌ Schema/table check failed: {e}")
            return False

    def test_specific_xml_query(self):
        """Test the specific XML xpath query from your screenshot"""
        print("\n🎯 Testing your specific XML xpath query...")
        
        if not self.engine:
            print("❌ No database connection available")
            return False
        
        # Your exact query from the screenshot
        test_query = """
        select (xpath('//globexUserSignature/globexUserSignatureInfo/globexFirmID/text()',guses.XML))[1]::text as "GFID",
        (xpath('//globexUserSignature/globexUserSignatureInfo/idNumber/text()',guses.XML))[1]::text as "GUS_ID",
        (xpath('//globexUserSignature/globexUserSignatureInfo/exchange/text()',guses.XML))[1]::text as "Exchange"
        from pr01cosrs.ACTIVE_XML_DATA_STORE guses 
        where guses.namespace = 'GlobexUserSignature' and GUID='JUY3TALM2SSP'
        """
        
        try:
            with self.engine.connect() as connection:
                print("🔍 Executing your XML xpath query...")
                print("📝 Query:")
                print(f"   {test_query.strip()}")
                
                result = connection.execute(text(test_query))
                rows = result.fetchall()
                
                print(f"\n📊 Query Results ({len(rows)} records found):")
                if rows:
                    # Print column headers
                    columns = result.keys()
                    print(f"   Columns: {', '.join(columns)}")
                    
                    # Print each row
                    for i, row in enumerate(rows):
                        print(f"\n   Row {i+1}:")
                        for j, col in enumerate(columns):
                            value = row[j] if row[j] is not None else 'NULL'
                            print(f"     {col}: {value}")
                    
                    print("✅ XML xpath query executed successfully!")
                    return True
                else:
                    print("⚠️  Query executed but no records found for GUID='JUY3TALM2SSP'")
                    
                    # Let's try to find some sample records
                    print("\n🔍 Looking for sample records...")
                    sample_query = f"""
                    SELECT GUID, namespace, 
                           CASE 
                               WHEN LENGTH(XML::text) > 100 THEN LEFT(XML::text, 100) || '...'
                               ELSE XML::text
                           END as xml_sample
                    FROM {self.oscar_schema}.ACTIVE_XML_DATA_STORE 
                    WHERE namespace = 'GlobexUserSignature'
                    LIMIT 5
                    """
                    
                    sample_result = connection.execute(text(sample_query))
                    sample_rows = sample_result.fetchall()
                    
                    if sample_rows:
                        print(f"   Found {len(sample_rows)} sample GlobexUserSignature records:")
                        for row in sample_rows:
                            print(f"     GUID: {row[0]}")
                            print(f"     Namespace: {row[1]}")
                            print(f"     XML Sample: {row[2][:100]}...")
                            print()
                    else:
                        print("   No GlobexUserSignature records found")
                    
                    return True
                
        except Exception as e:
            print(f"❌ XML xpath query failed: {e}")
            print(f"   Error Type: {type(e).__name__}")
            
            # Check if it's an XML/xpath related error
            error_str = str(e).lower()
            if "xpath" in error_str:
                print("\n💡 XPath issue:")
                print("   1. Verify XML structure in the database")
                print("   2. Check if XML column contains valid XML")
                print("   3. Verify namespace and GUID values")
            elif "permission denied" in error_str:
                print("\n💡 Permission issue:")
                print("   1. Check if user has SELECT permissions on the table")
                print("   2. Verify schema access permissions")
            
            return False

    def test_alternative_queries(self):
        """Test alternative queries to understand data structure"""
        print("\n🔍 Testing alternative queries to understand data structure...")
        
        if not self.engine:
            print("❌ No database connection available")
            return False
        
        try:
            with self.engine.connect() as connection:
                
                # Test 1: Check namespaces
                print("📊 Checking available namespaces...")
                namespace_query = f"""
                SELECT namespace, COUNT(*) as record_count
                FROM {self.oscar_schema}.ACTIVE_XML_DATA_STORE 
                GROUP BY namespace
                ORDER BY record_count DESC
                LIMIT 10
                """
                
                result = connection.execute(text(namespace_query))
                namespaces = result.fetchall()
                
                if namespaces:
                    print("   Available namespaces:")
                    for ns in namespaces:
                        print(f"     {ns[0]}: {ns[1]:,} records")
                else:
                    print("   No namespaces found")
                
                # Test 2: Check recent GUIDs
                print("\n📊 Checking recent GUIDs...")
                guid_query = f"""
                SELECT GUID, namespace, 
                       CASE 
                           WHEN XML IS NULL THEN 'NULL'
                           WHEN LENGTH(XML::text) > 50 THEN LEFT(XML::text, 50) || '...'
                           ELSE XML::text
                       END as xml_preview
                FROM {self.oscar_schema}.ACTIVE_XML_DATA_STORE 
                ORDER BY GUID
                LIMIT 5
                """
                
                result = connection.execute(text(guid_query))
                guids = result.fetchall()
                
                if guids:
                    print("   Recent GUIDs:")
                    for guid in guids:
                        print(f"     GUID: {guid[0]}")
                        print(f"     Namespace: {guid[1]}")
                        print(f"     XML Preview: {guid[2]}")
                        print()
                
                # Test 3: Test basic XML structure
                print("📊 Testing XML structure...")
                xml_test_query = f"""
                SELECT GUID, 
                       CASE 
                           WHEN XML IS NULL THEN 'NULL'
                           WHEN XML::text = '' THEN 'EMPTY'
                           WHEN XML::text LIKE '<%' THEN 'XML'
                           ELSE 'OTHER'
                       END as xml_type,
                       LENGTH(XML::text) as xml_length
                FROM {self.oscar_schema}.ACTIVE_XML_DATA_STORE 
                LIMIT 5
                """
                
                result = connection.execute(text(xml_test_query))
                xml_info = result.fetchall()
                
                if xml_info:
                    print("   XML structure analysis:")
                    for info in xml_info:
                        print(f"     GUID: {info[0]} | Type: {info[1]} | Length: {info[2]}")
                
                return True
                
        except Exception as e:
            print(f"❌ Alternative queries failed: {e}")
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

def run_enhanced_test():
    """Run complete enhanced test suite"""
    start_time = time.time()
    
    try:
        # Initialize tester
        tester = EnhancedOSCARTester()
        
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
        
        # Step 4: Test OSCAR schema and tables
        if not tester.test_oscar_schema_and_tables():
            return False
        
        # Step 5: Test your specific XML query
        tester.test_specific_xml_query()
        
        # Step 6: Test alternative queries for understanding
        tester.test_alternative_queries()
        
        # Success summary
        end_time = time.time()
        print("\n🎉 All enhanced tests completed!")
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
    print("🚀 Starting Enhanced OSCAR Database Connection Test")
    print(f"👤 User: lakshya.vijay@cmegroup.com")
    print(f"🏥 Database: OSCAR on Google Cloud SQL")
    print(f"🎯 Testing specific XML xpath query")
    
    success = run_enhanced_test()
    
    if success:
        print("\n✅ Enhanced OSCAR database connectivity test PASSED!")
        print("🎯 Your XML xpath query has been tested!")
        print("🚀 Ready to proceed with full reconciliation application!")
    else:
        print("\n❌ Enhanced OSCAR database connectivity test FAILED!")
        print("🔧 Please fix the issues above before proceeding.")
    
    return 0 if success else 1

if __name__ == "__main__":
    sys.exit(main())