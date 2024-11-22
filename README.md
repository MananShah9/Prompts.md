const { Client } = require('pg');

(async () => {
  // Database connection details
  const client = new Client({
    user: 'AutoQlik',
    host: 'localhost',
    database: 'CXODashboard_db_dev',
    password: 'AutoClick',
    port: 4432,
  });

  try {
    // Connect to the database
    await client.connect();
    console.log('Connected to PostgreSQL database');

    // Query to get all table names
    const tablesQuery = `
      SELECT table_schema, table_name 
      FROM information_schema.tables
      WHERE table_type = 'BASE TABLE' AND table_schema NOT IN ('pg_catalog', 'information_schema');
    `;

    const tablesResult = await client.query(tablesQuery);
    const tables = tablesResult.rows;

    console.log(`Found ${tables.length} tables`);

    // Function to get DDL for a specific table
    const getTableDDL = async (schema, tableName) => {
      const ddlQuery = `
        SELECT 'CREATE TABLE ' || quote_ident(schemaname) || '.' || quote_ident(tablename) || E' (\n' ||
               array_to_string(array_agg('  ' || column_name || ' ' || data_type || 
                   COALESCE('(' || character_maximum_length || ')', '') || 
                   (CASE WHEN is_nullable = 'NO' THEN ' NOT NULL' ELSE '' END) || 
                   (CASE WHEN column_default IS NOT NULL THEN ' DEFAULT ' || column_default ELSE '' END)), E',\n') || E'\n);' AS ddl
        FROM (
          SELECT 
            schemaname, tablename, 
            column_name, data_type, character_maximum_length, is_nullable, column_default
          FROM information_schema.columns
          WHERE table_schema = $1 AND table_name = $2
          ORDER BY ordinal_position
        ) AS table_columns
        GROUP BY schemaname, tablename;
      `;
      const ddlResult = await client.query(ddlQuery, [schema, tableName]);
      return ddlResult.rows[0]?.ddl || `-- No DDL found for ${schema}.${tableName}`;
    };

    // Loop through all tables and fetch DDL
    for (const { table_schema, table_name } of tables) {
      const ddl = await getTableDDL(table_schema, table_name);
      console.log(`DDL for table ${table_schema}.${table_name}:\n`);
      console.log(ddl);
      console.log('\n--------------------------------------------------------\n');
    }
  } catch (err) {
    console.error('Error:', err);
  } finally {
    // Close the connection
    await client.end();
    console.log('Connection closed');
  }
})();
