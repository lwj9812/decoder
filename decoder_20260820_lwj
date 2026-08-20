package main

import (
	"bufio"
	"bytes"
	"encoding/json"
	"fmt"
	"math/bits"
	"os"
	"os/exec"
	"regexp"
	"sort"
	"strconv"
	"strings"
	"sync"
	"time"
)

type CommonLogHeader struct {
	Timestamp string          `json:"timestamp"`
	Level     string          `json:"level"`
	Message   string          `json:"message"`
	Type      string          `json:"type"`
	Body      json.RawMessage `json:"body"`
}

type DRAMECCBody struct {
	Channel           int    `json:"channel"`
	SubChannel        int    `json:"sub_channel"`
	DIMM              int    `json:"dimm"`
	Rank              int    `json:"rank"`
	CID               int    `json:"cid"`
	BankGroup         int    `json:"bank_group"`
	Bank              int    `json:"bank"`
	Row               int    `json:"row"`
	Column            int    `json:"column"`
	DIMMMfg           string `json:"dimm_mfg"`
	DIMMSN            string `json:"dimm_sn"`
	DIMMPN            string `json:"dimm_pn"`
	CorrectableCnt    int    `json:"correctable_cnt"`
	PersistCnt        int    `json:"persist_cnt"`
	TransientCnt      int    `json:"transient_cnt"`
	PhysAddress       string `json:"phys_address"`
	Temperature       int    `json:"temperature"`
	PowerMode         string `json:"power_mode"`
	ScrubberError     string `json:"scrubber_error"`
	PreReplay         string `json:"pre_replay"`
	Misc0             string `json:"misc0"`
	Misc1             string `json:"misc1"`
	BeatOffsetMask    string `json:"beat_offset_mask"`
	PhyLanesBitmask   string `json:"phy_lanes_bitmask"`
	SyndromeMask0     string `json:"syndrome_bitmask0"`
	SyndromeMask1     string `json:"syndrome_bitmask1"`
	PreReplayBitmask0 string `json:"pre_replay_bitmask0"`
	PreReplayBitmask1 string `json:"pre_replay_bitmask1"`
}

var (
	monitorStartTime time.Time
	ucBaseBootTime   time.Time // 💡 UC 커널 부팅 기준 시간
	seenLogs         = make(map[string]bool)

	// GC Optimizations
	cpuRegex     = regexp.MustCompile(`(?i)CPU(\d+)`)
	numbersRegex = regexp.MustCompile(`\d+`)

	logPool = sync.Pool{
		New: func() interface{} {
			return &CommonLogHeader{}
		},
	}
	bodyPool = sync.Pool{
		New: func() interface{} {
			return &DRAMECCBody{}
		},
	}
)

func main() {
	monitorStartTime = time.Now()
	
	// 💡 UC 자체의 /proc/uptime을 읽어, 서버가 켜진 정확한 캘린더 날짜/시간을 계산
	ucBaseBootTime = calculateBootTime()

	armToolPath := findArmTool()
	snToLocator := fetchDimmInfoAuto(armToolPath)
	printHeaderAndTable(snToLocator)

	args := os.Args[1:]
	if len(args) == 0 {
		args = []string{"--log", "-r", "--type", "el3"}
	}

	for {
		runMonitor(armToolPath, snToLocator, args)
		time.Sleep(1 * time.Second)
	}
}

func runMonitor(armToolPath string, snToLocator map[string]string, args []string) {
	var cmd *exec.Cmd

	stdbufPath, err := exec.LookPath("stdbuf")
	if err == nil {
		cmdArgs := append([]string{"-oL", armToolPath}, args...)
		cmd = exec.Command(stdbufPath, cmdArgs...)
	} else {
		cmd = exec.Command(armToolPath, args...)
	}

	stdout, err := cmd.StdoutPipe()
	if err != nil {
		return
	}

	if err := cmd.Start(); err != nil {
		return
	}

	scanner := bufio.NewScanner(stdout)
	for scanner.Scan() {
		line := scanner.Bytes()
		if len(line) == 0 {
			continue
		}

		logPtr := logPool.Get().(*CommonLogHeader)
		if err := json.Unmarshal(line, logPtr); err != nil {
			logPool.Put(logPtr)
			continue
		}

		cacheKey := logPtr.Timestamp + "_" + logPtr.Type
		if seenLogs[cacheKey] {
			logPtr.Body = nil
			logPool.Put(logPtr)
			continue
		}
		seenLogs[cacheKey] = true

		if len(seenLogs) > 10000 {
			seenLogs = make(map[string]bool)
		}

		if logPtr.Type == "dram_ecc_event" {
			printDRAMEvent(*logPtr, snToLocator)
		}
		
		logPtr.Body = nil
		logPool.Put(logPtr)
	}
	cmd.Wait()
}

func parseDQInfo(hexMask string) (int, []int) {
	hexMask = strings.TrimPrefix(hexMask, "0x")
	val, err := strconv.ParseUint(hexMask, 16, 64)
	if err != nil || val == 0 {
		return 0, nil
	}

	dqCount := bits.OnesCount64(val)
	var activeLanes []int
	for i := 0; i < 64; i++ {
		if (val & (1 << i)) != 0 {
			activeLanes = append(activeLanes, i)
		}
	}
	return dqCount, activeLanes
}

func findArmTool() string {
	paths := []string{"/usr/local/bin/arm_tool", "/usr/bin/arm_tool", "/opt/bin/arm_tool"}
	for _, p := range paths {
		if _, err := os.Stat(p); err == nil {
			return p
		}
	}
	if path, err := exec.LookPath("arm_tool"); err == nil {
		return path
	}
	return "arm_tool"
}

// 💡 UC(호스트 OS)의 /proc/uptime을 기반으로 부팅 기준 시점을 도출
func calculateBootTime() time.Time {
	content, err := os.ReadFile("/proc/uptime")
	if err == nil {
		parts := strings.Fields(string(content))
		if len(parts) > 0 {
			uptimeSec, err := strconv.ParseFloat(parts[0], 64)
			if err == nil {
				uptimeDuration := time.Duration(uptimeSec * float64(time.Second))
				// 현재 시간에서 Uptime을 빼서 부팅 시점 도출
				return time.Now().Add(-uptimeDuration)
			}
		}
	}
	// /proc/uptime을 못 읽으면 어쩔 수 없이 현재 시간을 Base로 사용 (비상용)
	return time.Now()
}

func getSystemInfo() (string, string, string) {
	manufacturer, vendor, cpus := "Amazon EC2", "ARM", "193"
	if b, err := os.ReadFile("/sys/class/dmi/id/sys_vendor"); err == nil && strings.TrimSpace(string(b)) != "" {
		manufacturer = strings.TrimSpace(string(b))
	}
	out, err := exec.Command("lscpu").Output()
	if err == nil {
		lines := strings.Split(string(out), "\n")
		for _, line := range lines {
			if strings.HasPrefix(line, "Vendor ID:") {
				vendor = strings.TrimSpace(strings.TrimPrefix(line, "Vendor ID:"))
			}
		}
	}
	return manufacturer, vendor, cpus
}

func fetchDimmInfoAuto(armToolPath string) map[string]string {
	mapping := make(map[string]string)
	out, err := exec.Command(armToolPath, "--dimm-info").Output()
	if err != nil {
		return mapping
	}

	scanner := bufio.NewScanner(bytes.NewReader(out))
	currentLocator := ""
	for scanner.Scan() {
		line := strings.TrimSpace(scanner.Text())
		if strings.HasPrefix(line, "Locator:") {
			currentLocator = strings.TrimSpace(strings.TrimPrefix(line, "Locator:"))
		} else if strings.HasPrefix(line, "SN:") {
			sn := strings.TrimSpace(strings.TrimPrefix(line, "SN:"))
			sn = strings.ToUpper(strings.ReplaceAll(sn, "-", ""))
			if currentLocator != "" && sn != "" {
				mapping[sn] = currentLocator
			}
		}
	}
	return mapping
}

type DimmEntry struct {
	SN      string
	Locator string
}

func extractNumbers(s string) []int {
	matches := numbersRegex.FindAllString(s, -1)
	var nums []int
	for _, m := range matches {
		n, _ := strconv.Atoi(m)
		nums = append(nums, n)
	}
	return nums
}

func printHeaderAndTable(mapping map[string]string) {
	manufacturer, vendor, cpus := getSystemInfo()

	fmt.Println("\n-------------------------------------------------")
	fmt.Printf("| %-20s | %-20s |\n", "Locator", "serial")
	fmt.Println("-------------------------------------------------")

	if len(mapping) == 0 {
		fmt.Printf("| %-20s | %-20s |\n", "No mapping data found", "-")
	} else {
		var entries []DimmEntry
		for sn, loc := range mapping {
			entries = append(entries, DimmEntry{SN: sn, Locator: loc})
		}
		sort.Slice(entries, func(i, j int) bool {
			numsI := extractNumbers(entries[i].Locator)
			numsJ := extractNumbers(entries[j].Locator)
			for k := 0; k < len(numsI) && k < len(numsJ); k++ {
				if numsI[k] != numsJ[k] {
					return numsI[k] < numsJ[k]
				}
			}
			return entries[i].Locator < entries[j].Locator
		})
		for _, entry := range entries {
			fmt.Printf("| %-20s | %-20s |\n", entry.Locator, entry.SN)
		}
	}
	fmt.Println("=================================================")
	fmt.Printf("server manufacturer : %s\n", manufacturer)
	fmt.Printf("Vendor ID           : %s\n", vendor)
	fmt.Println("**** This tool provides real-time monitoring of ARM EL3 hardware errors,")
	fmt.Println("**** mapping ECC memory faults to physical DIMM topologies for rapid diagnostics.")
	fmt.Println("=================================================")
	fmt.Printf("Program Name        : ARM EL3 Memory Fault Detector\n")
	fmt.Printf("start time          : %s\n", monitorStartTime.Format("2006-01-02 15:04:05"))
	fmt.Printf("uc boot time        : %s\n", ucBaseBootTime.Format("2006-01-02 15:04:05"))
	fmt.Println("=================================================")
	fmt.Printf("The number of cpus  : %s\n", cpus)
	fmt.Println("=================================================")
}

func parseTimestamp(ts string, baseBootTime time.Time) string {
	var logMicro int64
	var err error

	ts = strings.TrimSpace(ts)
	if strings.HasPrefix(ts, "0x") || strings.HasPrefix(ts, "0X") {
		if val, parseErr := strconv.ParseInt(ts[2:], 16, 64); parseErr == nil {
			logMicro = val
		} else {
			err = parseErr
		}
	} else {
		if val, parseErr := strconv.ParseInt(ts, 10, 64); parseErr == nil {
			logMicro = val
		} else {
			err = parseErr
		}
	}

	if err == nil {
		realLogTime := baseBootTime.Add(time.Duration(logMicro) * time.Microsecond)
		return realLogTime.Format("2006-01-02 15:04:05")
	}
	
	return time.Now().Format("2006-01-02 15:04:05")
}

func parseMisc0(misc0 string) string {
	misc0 = strings.TrimPrefix(misc0, "0x")
	misc0 = strings.TrimPrefix(misc0, "0X")
	val, err := strconv.ParseUint(misc0, 16, 64)
	if err != nil {
		return "N"
	}
	// ARM DDI 0587: ERR<n>STATUS Bit 31 is OF (Overflow)
	if (val & (1 << 31)) != 0 {
		return "Y"
	}
	return "N"
}

func determineErrType(isUE bool, dqCount int, isPreReplay bool, persistCnt, correctableCnt int) string {
	if isUE {
		return "CRITICAL UCE"
	}
	if dqCount >= 4 {
		return "[PFA: Single Device/Chip Fault (Critical)]"
	}
	if isPreReplay {
		return "[PFA: Link/RCD Interface Signal Noise]"
	}
	
	if correctableCnt >= 10 {
		pr := float64(persistCnt) / float64(correctableCnt)
		if pr >= 0.8 {
			return "[PFA: Hard Stuck-at Fault (Action: Page Offline)]"
		}
	}

	if correctableCnt >= 1000 {
		return "[PFA: CE Storm / Imminent UCE Risk]"
	}

	return "CE error"
}

func printDRAMEvent(log CommonLogHeader, mapping map[string]string) {
	bodyPtr := bodyPool.Get().(*DRAMECCBody)
	defer bodyPool.Put(bodyPtr)

	if err := json.Unmarshal(log.Body, bodyPtr); err != nil {
		return
	}
	body := *bodyPtr

	cleanSN := strings.ToUpper(strings.ReplaceAll(body.DIMMSN, "-", ""))
	locator := mapping[cleanSN]
	if locator == "" {
		locator = "UNKNOWN_SLOT"
	}

	// 💡 가장 완벽하고 오차 없는 타임스탬프 계산법
	nowStr := parseTimestamp(log.Timestamp, ucBaseBootTime)

	socketNum := "0"
	if len(locator) > 3 && strings.ToUpper(locator[:3]) == "CPU" {
		match := cpuRegex.FindStringSubmatch(locator)
		if len(match) > 1 {
			socketNum = match[1]
		}
	}

	scStr := "A"
	if body.SubChannel == 1 {
		scStr = "B"
	}

	dqCount, _ := parseDQInfo(body.PhyLanesBitmask)

	beatOffset := strings.TrimPrefix(body.BeatOffsetMask, "0x")
	if beatOffset == "" {
		beatOffset = "0"
	}

	rowHex := strings.ToUpper(fmt.Sprintf("%04Xh", body.Row))
	colHex := strings.ToUpper(fmt.Sprintf("%03Xh", body.Column))
	statusHex := strings.TrimPrefix(body.Misc0, "0x") + "h"
	parityHex := strings.TrimPrefix(body.SyndromeMask0, "0x") + "h"
	retryLogHex := strings.TrimPrefix(body.PreReplayBitmask0, "0x") + "h"

	patSpr := "N"
	isUE := strings.Contains(strings.ToLower(log.Message), "uncorrectable") || strings.ToUpper(log.Level) == "FATAL"
	isScrub := strings.ToLower(body.ScrubberError) == "true"
	isPreReplay := strings.ToLower(body.PreReplay) == "true"

	if isScrub {
		patSpr = "Y"
	}

	errType := determineErrType(isUE, dqCount, isPreReplay, body.PersistCnt, body.CorrectableCnt)
	if isScrub && errType == "CE error" {
		errType = "Patrol scrubbing/sparing error"
	}

	retryReg := "Invalid"
	if isPreReplay {
		retryReg = "Valid"
	}
	
	overflowFlag := parseMisc0(body.Misc0)

	extraInfo := cleanSN
	if extraInfo == "" || extraInfo == "UNKNOWN_SLOT" {
		extraInfo = "None"
	}

	level := strings.ToUpper(log.Level)
	if level == "" {
		level = "ERROR"
	}

	summaryType := "Correctable"
	if isUE {
		summaryType = "Uncorrectable"
	}

	ceCountStr := fmt.Sprintf("%d(0x%x)", body.CorrectableCnt, body.CorrectableCnt)

	var dqLanes []string
	for _, l := range activeLanes {
		dqLanes = append(dqLanes, fmt.Sprintf("DQ%d", l))
	}
	dqLanesStr := "None"
	if len(dqLanes) > 0 {
		dqLanesStr = "[" + strings.Join(dqLanes, ", ") + "]"
	}

	powerMode := body.PowerMode
	if powerMode == "" {
		powerMode = "Unknown"
	}

	scrubberStr := "False"
	if isScrub {
		scrubberStr = "True"
	}

	preReplayStr := "False"
	if isPreReplay {
		preReplayStr = "True"
	}

	fmt.Printf("[%s] %s | DRAM %s ECC event summary | %s (%s) | Addr: %s | CE: %s | DQ Err Cnt: %d\n",
		nowStr, level, summaryType, locator, cleanSN, body.PhysAddress, ceCountStr, dqCount)

	fmt.Printf("           └ [Topology] Socket:%s Ch:%d SubCh:%d(%s) DIMM:%d Rank:%d CID:%d BG:%d Bank:%d Row:%d(%s) Col:%d(%s)\n",
		socketNum, body.Channel, body.SubChannel, scStr, body.DIMM, body.Rank, body.CID, body.BankGroup, body.Bank, body.Row, rowHex, body.Column, colHex)

	fmt.Printf("           └ [DQ Signal] Active DQ Errors: %d | Affected Lanes: %s | BeatOffset: %s | Parity: %s\n",
		dqCount, dqLanesStr, beatOffset, parityHex)

	fmt.Printf("           └ [Error Cnt] CE: %d(0x%x) | Persist: %d(0x%x) | Transient: %d(0x%x) | Overflow: %s\n",
		body.CorrectableCnt, body.CorrectableCnt, body.PersistCnt, body.PersistCnt, body.TransientCnt, body.TransientCnt, overflowFlag)

	fmt.Printf("           └ [Status]   Temp:%dC | Power:%s | Scrubber:%s | PreReplay:%s | RetryReg:%s | RetryLog:%s | PatSpr:%s | StatusHex:%s | ErrType: %s\n",
		body.Temperature, powerMode, scrubberStr, preReplayStr, retryReg, retryLogHex, patSpr, statusHex, errType)
}