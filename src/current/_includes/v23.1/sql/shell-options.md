- To view option descriptions and how they are currently set, use `\set` without any options.
- To enable or disable an option, use `\set <option> <value>` or `\unset <option> <value>`. You can also use the form `<option>=<value>`.
- If an option accepts a boolean value:
    - `\set <option>` without `<value>` is equivalent to `\set <option> true`, and `\unset <option>` without `<value>` is equivalent to `\set <option> false`.
    - `on`, `yes`, and `1` are aliases for `true`, and `off`, `no`, and `0` are aliases for `false`.

Client Options | Description
---------------|------------
<a name="sql-option-auto-trace"></a> `auto_trace` | For every statement executed, the shell also produces the trace for that statement in a separate result below. A trace is also produced in case the statement produces a SQL error.<br><br>**Default:** `off`<br><br>To enable this option, run `\set auto_trace on`.
<a name="sql-option-border"></a> `border` |  Display a border around the output of the SQL statement when using the `table` display format. Set the level of borders using `border=<level>` to configure how many borders and lines are in the output, where `<level>` is an integer between 0 and 3. The higher the integer, the more borders and lines are in the output. <br>A level of `0` shows the output with no outer lines and no row line separators. <br>A level of `1` adds row line separators. A level of `2` adds an outside border and no row line separators. A level of `3` adds both an outside border and row line separators. <br><br>**Default:** `0` <br><br>To change this option, run `\set border=<level>`. [See an example]({% link {{ page.version.version }}/cockroach-sql.md %}#show-borders-around-the-statement-output-within-the-sql-shell).
<a name="sql-option-check-syntax"></a> `check_syntax` | Validate SQL syntax. This ensures that a typo or mistake during user entry does not inconveniently abort an ongoing transaction previously started from the interactive shell.<br /><br />**Default:** `true` for [interactive sessions]({% link {{ page.version.version }}/cockroach-sql.md %}#session-and-output-types); `false` otherwise.<br><br>To disable this option, run `\unset check_syntax`.
<a name="sql-option-display-format"></a> `display_format` | How to display table rows printed within the interactive SQL shell. Possible values: `tsv`, `csv`, `table`, `raw`, `records`, `sql`, `html`.<br><br>**Default:** `table` for sessions that [output on a terminal]({% link {{ page.version.version }}/cockroach-sql.md %}#session-and-output-types); `tsv` otherwise<br /><br />To change this option, run `\set display_format <format>`. [See an example]({% link {{ page.version.version }}/cockroach-sql.md %}#make-the-output-of-show-statements-selectable).
<a name="sql-option-echo"></a> `echo` | Reveal the SQL statements sent implicitly by the SQL shell.<br><br>**Default:** `false`<br><br>To enable this option, run `\set echo`. [See an example]({% link {{ page.version.version }}/cockroach-sql.md %}#reveal-the-sql-statements-sent-implicitly-by-the-command-line-utility).
<a name="sql-option-errexit"></a> `errexit` | Exit the SQL shell upon encountering an error.<br /><br />**Default:** `false` for [interactive sessions]({% link {{ page.version.version }}/cockroach-sql.md %}#session-and-output-types); `true` otherwise<br><br>To enable this option, run `\set errexit`.
<a name="sql-option-prompt1"></a> `prompt1` | Customize the interactive prompt within the SQL shell. See [Customizing the prompt](#customizing-the-prompt) for information on the available prompt variables.
<a name="sql-option-show-times"></a> `show_times` | Reveal the time a query takes to complete. Possible values:<br><ul><li>`execution` time refers to the time taken by the SQL execution engine to execute the query.</li><li>`network` time refers to the network latency between the server and the SQL client command.</li><li>`other` time refers to all other forms of latency affecting the total query completion time, including query planning.</li></ul>**Default:** `true`<br><br>To disable this option, run `\unset show_times`.

#### Customizing the prompt

The `\set prompt1` option allows you to customize the interactive prompt in the SQL shell. Use the following prompt variables to set a custom prompt.

Prompt variable | Description
----------------|------------
`%>` | The port of the node you are connected to.
`%/` | The current database name.
`%M` | The fully qualified host name and port of the node.
`%m` | The fully qualified host name of the node.
`%n` | The username of the connected SQL user.
`%x` | The transaction status of the current statement.

For example, to change the prompt to just the user, host, and database:

{% include_cached copy-clipboard.html %}
~~~ sql
\set prompt1 %n@%m/%/
~~~

~~~
maxroach@blue-dog-595.g95.cockroachlabs.cloud/defaultdb>
~~~
