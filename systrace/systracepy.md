# systrace.py

- systrace.py

```py
def main():
  options, categories = parse_options(sys.argv)
  agents = create_agents(options, categories)

  if not agents:
    dirs = DEFAULT_AGENT_DIR
    if options.agent_dirs:
      dirs += ',' + options.agent_dirs
    print >> sys.stderr, ('No systrace agent is available in directories |%s|.'
                          % dirs)
    sys.exit(1)

  for a in agents:
    # 啟動　atrace　= agents/atrace_agent.py
    a.start()

  for a in agents:
    # 收集資料
    a.collect_result()
    if not a.expect_trace():
      # Nothing more to do.
      return

  script_dir = os.path.dirname(os.path.abspath(sys.argv[0]))
  write_trace_html(options.output_file, script_dir, agents)

```


- 啟動　atrace　= agents/atrace_agent.py

```py
# tracer_args 內容　['adb', 'shell', 'atrace -z -t 100 sched gfx view wm memreclaim ; ps -t']

def start(self):
tracer_args = self._construct_trace_command()
try:
self._adb = subprocess.Popen(tracer_args, stdout=subprocess.PIPE,
                           stderr=subprocess.PIPE)
except OSError as error:
print >> sys.stderr, (
  'The command "%s" failed with the following error:' %
  ' '.join(tracer_args))
print >> sys.stderr, '    ', error
sys.exit(1)
```

- collect_result

```py
def collect_result(self):
trace_data = self._collect_trace_data();
if self._expect_trace:
    # 資料作處理
    self._trace_data = self._preprocess_trace_data(trace_data);
```
