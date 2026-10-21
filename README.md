
package com.example.oracle;

import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.regex.Pattern;

import org.springframework.ai.tool.annotation.Tool;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Component;

@Component
public class OracleTools {

    private final JdbcTemplate jdbc;

    private static final int ROW_LIMIT = 20;

    private static final Pattern IDENTIFIER =
            Pattern.compile("[A-Za-z][A-Za-z0-9_$#]{0,127}");

    public OracleTools(JdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    // 1. List tables
    @Tool(description = "List all tables in the current Oracle schema.")
    public List<Map<String, Object>> listTables() {
        return jdbc.queryForList("""
            SELECT table_name
            FROM user_tables
            ORDER BY table_name
            """);
    }

    // 2. Read table schema
    @Tool(description = "Get column names, data types and nullable status for an Oracle table.")
    public List<Map<String, Object>> getTableSchema(String tableName) {
        String table = validateIdentifier(tableName);

        return jdbc.queryForList("""
            SELECT column_name, data_type, data_length,
                   data_precision, data_scale, nullable
            FROM user_tab_columns
            WHERE table_name = ?
            ORDER BY column_id
            """, table);
    }

    // 3. Find tables containing a column
    @Tool(description = "Find every table in the current Oracle schema that contains the given column name.")
    public List<Map<String, Object>> findTablesByColumn(
            String columnName) {

        String column = validateIdentifier(columnName);

        return jdbc.queryForList("""
            SELECT table_name, column_name, data_type
            FROM user_tab_columns
            WHERE column_name = ?
            ORDER BY table_name
            """, column);
    }

    // 4. Inspect distinct column values
    @Tool(description = "Get up to 20 distinct non-null values from a specified table column.")
    public List<Map<String, Object>> getColumnValues(
            String tableName, String columnName) {

        String table = validateIdentifier(tableName);
        String column = validateColumn(table, columnName);

        String sql = """
            SELECT DISTINCT "%s" AS COLUMN_VALUE
            FROM "%s"
            WHERE "%s" IS NOT NULL
              AND ROWNUM <= %d
            """.formatted(column, table, column, ROW_LIMIT);

        return jdbc.queryForList(sql);
    }

    // 5. Search all tables for a column/value pair
    @Tool(description =
        "Find rows matching a column name and value across every table " +
        "in the current Oracle schema. Automatically discovers tables " +
        "containing the column and returns matching rows with table names. " +
        "Use for questions such as find CUSTOMER_ID 1001.")
    public Map<String, Object> findRowsByColumnValue(
            String columnName, String value) {

        String column = validateIdentifier(columnName);

        List<Map<String, Object>> tables =
                findTablesByColumn(column);

        List<Map<String, Object>> matches = new ArrayList<>();

        for (Map<String, Object> tableInfo : tables) {
            String table = tableInfo.get("TABLE_NAME").toString();

            String sql = """
                SELECT *
                FROM "%s"
                WHERE "%s" = ?
                  AND ROWNUM <= %d
                """.formatted(table, column, ROW_LIMIT);

            try {
                List<Map<String, Object>> rows =
                        jdbc.queryForList(sql, value);

                if (!rows.isEmpty()) {
                    Map<String, Object> result =
                            new LinkedHashMap<>();

                    result.put("tableName", table);
                    result.put("matchingRows", rows);

                    matches.add(result);
                }
            } catch (org.springframework.dao.DataAccessException ex) {
                // A table may have a different datatype or conversion rule.
                // Continue searching other tables.
            }
        }

        Map<String, Object> response = new LinkedHashMap<>();
        response.put("columnName", column);
        response.put("searchValue", value);
        response.put("tablesChecked", tables.size());
        response.put("matches", matches);

        return response;
    }

    // 6. Execute a read-only SELECT query
    @Tool(description =
        "Execute a single read-only SELECT query against Oracle. " +
        "Use actual table and column names discovered from schema tools. " +
        "Results are limited to 20 rows. Do not use DML or DDL.")
    public List<Map<String, Object>> executeSelectQuery(String query) {

        if (query == null || query.isBlank()) {
            throw new IllegalArgumentException("Query is required");
        }

        String sql = query.trim();

        if (sql.endsWith(";")) {
            sql = sql.substring(0, sql.length() - 1).trim();
        }

        if (!sql.regionMatches(true, 0, "SELECT ", 0, 7)
                || sql.contains(";")
                || sql.contains("--")
                || sql.contains("/*")
                || sql.contains("*/")
                || Pattern.compile(
                    "(?i)\\b(INSERT|UPDATE|DELETE|MERGE|DROP|ALTER|" +
                    "TRUNCATE|GRANT|REVOKE|COMMIT|ROLLBACK|CALL|" +
                    "EXECUTE|FOR\\s+UPDATE)\\b")
                    .matcher(sql).find()) {
            throw new IllegalArgumentException(
                    "Only a single read-only SELECT query is allowed");
        }

        String limitedSql =
                "SELECT * FROM (" + sql + ") WHERE ROWNUM <= " + ROW_LIMIT;

        return jdbc.queryForList(limitedSql);
    }

    private String validateIdentifier(String input) {
        if (input == null ||
                !IDENTIFIER.matcher(input).matches()) {
            throw new IllegalArgumentException(
                    "Invalid table or column identifier");
        }

        return input.toUpperCase();
    }

    private String validateColumn(String table, String column) {
        String name = validateIdentifier(column);

        Integer count = jdbc.queryForObject("""
            SELECT COUNT(*)
            FROM user_tab_columns
            WHERE table_name = ?
              AND column_name = ?
            """, Integer.class, table, name);

        if (count == null || count == 0) {
            throw new IllegalArgumentException(
                    "Column does not exist in table " + table);
        }

        return name;
    }
}
