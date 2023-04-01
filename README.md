
Beats usa esse processor

https://github.com/elastic/elastic-agent-system-metrics/blob/main/metric/system/cpu/cpu.go

Tem uma lib que retorna dados específicos pra qualquer plataforma

https://github.com/elastic/gosigar

# Como o Beats faz isso usando arquivos?

https://man7.org/linux/man-pages/man5/proc.5.html

Ele usa o module `system` do MetricBeat

https://github.com/elastic/beats/blob/main/metricbeat/module/system/cpu/cpu.go

```go
// Fetch fetches CPU metrics from the OS.
func (m *MetricSet) Fetch(r mb.ReporterV2) error {
    // aqui ele  chama o cara que alimenta a variavel sample
	sample, err := m.cpu.Fetch()
	if err != nil {
		return errors.Wrap(err, "failed to fetch CPU times")
	}

	event, err := sample.Format(m.opts)
	if err != nil {
		return errors.Wrap(err, "error formatting metrics")
	}
	event.Put("cores", sample.CPUCount())

	//generate the host fields here, since we don't want users disabling it.
	hostEvent, err := sample.Format(metrics.MetricOpts{NormalizedPercentages: true})
	if err != nil {
		return errors.Wrap(err, "error creating host fields")
	}
	hostFields := mapstr.M{}
	err = copyFieldsOrDefault(hostEvent, hostFields, "total.norm.pct", "host.cpu.usage", 0)
	if err != nil {
		return errors.Wrap(err, "error fetching normalized CPU percent")
	}

	r.Event(mb.Event{
		RootFields:      hostFields,
		MetricSetFields: event,
	})

	return nil
}

```
Esse componente é que carrega os dados via arquivo **/proc/stat**  e faz o parse

https://github.com/elastic/elastic-agent-system-metrics/blob/e28f1d367a927261c8020d832f0df6b390fb90ce/metric/cpu/metrics_procfs_common.go

https://github.com/elastic/elastic-agent-system-metrics/blob/e28f1d367a927261c8020d832f0df6b390fb90ce/metric/cpu/metrics_procfs_common.go#L116


```go

func Get(procfs resolve.Resolver) (CPUMetrics, error) {
	path := procfs.ResolveHostFS("/proc/stat")
	fd, err := os.Open(path)
	defer func() {
		_ = fd.Close()
	}()
	if err != nil {
		return CPUMetrics{}, fmt.Errorf("error opening file %s: %w", path, err)
	}

	metrics, err := scanStatFile(bufio.NewScanner(fd))
	if err != nil {
		return CPUMetrics{}, fmt.Errorf("scanning stat file: %w", err)
	}

	cpuInfoPath := procfs.ResolveHostFS("/proc/cpuinfo")
	cpuInfoFd, err := os.Open(cpuInfoPath)
	if err != nil {
		return CPUMetrics{}, fmt.Errorf("opening '%s': %w", cpuInfoPath, err)
	}
	defer cpuInfoFd.Close()

	cpuInfo, err := scanCPUInfoFile(bufio.NewScanner(cpuInfoFd))
	metrics.CPUInfo = cpuInfo

	return metrics, err
}

// parser linhas de /proc/stats
func parseCPULine(line string) (CPU, error) {

	var errs multierror.Errors
	tryParseUint := func(name, field string) (v opt.Uint) {
		u, err := touint(field)
		if err != nil {
			errs = append(errs, fmt.Errorf("failed to parse %v: %s", name, field))
		} else {
			v = opt.UintWith(u)
		}
		return v
	}

	cpuData := CPU{}
	fields := strings.Fields(line)

	cpuData.User = tryParseUint("user", fields[1])
	cpuData.Nice = tryParseUint("nice", fields[2])
	cpuData.Sys = tryParseUint("sys", fields[3])
	cpuData.Idle = tryParseUint("idle", fields[4])
	cpuData.Wait = tryParseUint("wait", fields[5])
	cpuData.Irq = tryParseUint("irq", fields[6])
	cpuData.SoftIrq = tryParseUint("softirq", fields[7])
	cpuData.Stolen = tryParseUint("stolen", fields[8])

	return cpuData, errs.Err()
}

// parser linhas de /proc/cpuinfo
func cpuinfoScanner(scanner *bufio.Scanner) ([]CPUInfo, error) {
	cpuInfos := []CPUInfo{}
	current := CPUInfo{}
	// On my tests the order the cores appear on /proc/cpuinfo
	// is the same as on /proc/stats, this means it matches our
	// current 'system.core.id' metric. This information
	// is also the same as the 'processor' line on /proc/cpuinfo.
	coreID := 0
	for scanner.Scan() {
		line := scanner.Text()
		split := strings.Split(line, ":")
		if len(split) != 2 {
			// A blank line its a separation between CPUs
			// even the last CPU contains one blank line at the end
			cpuInfos = append(cpuInfos, current)
			current = CPUInfo{}
			coreID++

			continue
		}

		k, v := split[0], split[1]
		k = strings.TrimSpace(k)
		v = strings.TrimSpace(v)
		switch k {
		case "model":
			current.ModelNumber = v
		case "model name":
			current.ModelName = v
		case "physical id":
			id, err := strconv.Atoi(v)
			if err != nil {
				return []CPUInfo{}, fmt.Errorf("parsing physical ID: %w", err)
			}
			current.PhysicalID = id
		case "core id":
			id, err := strconv.Atoi(v)
			if err != nil {
				return []CPUInfo{}, fmt.Errorf("parsing core ID: %w", err)
			}
			current.CoreID = id

        // isso eh o cpu ticks
		case "cpu MHz":
			mhz, err := strconv.ParseFloat(v, 64)
			if err != nil {
				return []CPUInfo{}, fmt.Errorf("parsing CPU %d Mhz: %w", coreID, err)
			}
			current.Mhz = mhz
		}
	}

	return cpuInfos, nil
}
```

numero de CPUs
https://github.com/elastic/elastic-agent-system-metrics/blob/e28f1d367a927261c8020d832f0df6b390fb90ce/metric/system/numcpu/cpu_linux.go#L62
```go
func getCPU() (int, bool, error) {

	// These are the files that LSCPU looks for
	// This will report online CPUs, which are are the logical CPUS
	// that are currently online and scheduleable by the system.
	// Some users may expect a "present" count, which reflects what
	// CPUs are available to the OS, online or off.
	// These two values will only differ in cases where CPU hotplugging is in affect.
	// This env var swaps between them.
	_, isPresent := os.LookupEnv("LINUX_CPU_COUNT_PRESENT")
	var cpuPath = "/sys/devices/system/cpu/online"
	if isPresent {
		cpuPath = "/sys/devices/system/cpu/present"
	}

	rawFile, err := ioutil.ReadFile(cpuPath)
	// if the file doesn't exist, assume it's a support issue and not a bug
	if errors.Is(err, os.ErrNotExist) {
		return -1, false, nil
	}
	if err != nil {
		return -1, false, fmt.Errorf("error reading file %s: %w", cpuPath, err)
	}

	cpuCount, err := parseCPUList(string(rawFile))
	if err != nil {
		return -1, false, fmt.Errorf("error parsing file %s: %w", cpuPath, err)
	}
	return cpuCount, true, nil
}


// parse the weird list files we get from sysfs
func parseCPUList(raw string) (int, error) {

	listPart := strings.Split(raw, ",")
	count := 0
	for _, v := range listPart {
		if strings.Contains(v, "-") {
			rangeC, err := parseCPURange(v)
			if err != nil {
				return 0, fmt.Errorf("error parsing line %s: %w", v, err)
			}
			count = count + rangeC
		} else {
			count++
		}
	}
	return count, nil
}

func parseCPURange(cpuRange string) (int, error) {
	var first, last int
	_, err := fmt.Sscanf(cpuRange, "%d-%d", &first, &last)
	if err != nil {
		return 0, fmt.Errorf("error reading from range %s: %w", cpuRange, err)
	}

	return (last - first) + 1, nil
}
```