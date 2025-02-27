package com.cognizant.collector.jirazephyr.component;

import com.cognizant.collector.jirazephyr.beans.zephyrScale.*;
import com.cognizant.collector.jirazephyr.client.ZephyrClient;
import com.cognizant.collector.jirazephyr.service.TestCaseService;
import com.cognizant.collector.jirazephyr.service.ZephyrScaleTestRunService;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;
import java.util.ArrayList;
import java.util.List;
import java.util.Optional;
import java.util.stream.Collectors;
import static com.cognizant.collector.jirazephyr.constants.Constant.PAGE_STARTS_AT;
import static com.cognizant.collector.jirazephyr.constants.Constant.RESULTS_PER_PAGE;

@Slf4j
@Component
public class ZephyrScaleTestRunComponent {

    @Value("${zephyrServer.token}")
    public String token;

    @Autowired
    private ZephyrClient zephyrScaleClient;

    @Autowired
    private ZephyrScaleTestRunService testRunService;

    @Autowired
    private TestCaseService testCaseService;

    public void getZephyrScaleTestRun(String projectKey) {
        System.out.println("Entering into ZephyrScale TestRun Collection");
        int maxResults = RESULTS_PER_PAGE;
        int startAt = PAGE_STARTS_AT;
        int totalSavedCount=0;
        boolean isLast = false;
        do {
            TestExecutionResponse testExecutionInfo = zephyrScaleClient.getTestExecutions(token, projectKey, maxResults, startAt);
            System.out.println("Test Execution Data: " + testExecutionInfo);
            Project project = zephyrScaleClient.getProject(token, projectKey);
            System.out.println("Project id: " + project.getId());
            log.info("Project Key - " + projectKey + " with ProjectId - " + project.getId());
            log.info("Zephyrscale data starting from " + startAt + " ending with " + (startAt + maxResults));
            List<ZephyrTestRun> zephyrScaleTestExecutions = testExecutionInfo.getValues().stream()
                    .filter(zephyrTestRun -> project.getId() == (zephyrTestRun.getProject().getId()))
                    .map(zephyrTestRun -> {
                        ZephyrTestRun zephyrScaleTestExecution = new ZephyrTestRun();
                        zephyrScaleTestExecution.setId(zephyrTestRun.getId());



                        // Fetching test case details using self URL
                        if (zephyrTestRun.getTestCase() != null) {
                            log.info("Test case is present for test execution: {}", zephyrTestRun.getKey());
                            if (zephyrTestRun.getTestCase().getSelf() != null) {
                                log.info("Test case self URL: {}", zephyrTestRun.getTestCase().getSelf());
                                TestCase testCaseInfo = testCaseService.getTestCaseInfo(zephyrTestRun.getTestCase().getSelf(), token);
                                log.info("Test case info: {}", testCaseInfo);
                                zephyrScaleTestExecution.setTestCaseId(testCaseInfo.getId());
                                zephyrScaleTestExecution.setTestCaseName(testCaseInfo.getName());
                                zephyrScaleTestExecution.setType(testCaseInfo.getCustomFields().getType());
                                zephyrScaleTestExecution.setTestcaseKey(testCaseInfo.getKey());
                            } else {
                                log.warn("Test case self URL is null for test execution: {}", zephyrTestRun.getKey());
                            }
                        } else {
                            log.warn("Test case is null for test execution: {}", zephyrTestRun.getKey());
                        }

                        // Environment details
                        Environment environmentDetails = null;
                        if (zephyrTestRun.getEnvironment() != null) {
                            environmentDetails = zephyrScaleClient.getEnvironments(token, String.valueOf(zephyrTestRun.getEnvironment().getId()));
                            if (environmentDetails == null) {
                                log.warn("Environment details are null for environment ID: {}", zephyrTestRun.getEnvironment().getId());
                            }
                        } else {
                            log.warn("Environment is null for test execution: {}", zephyrTestRun.getKey());
                        }
                        zephyrScaleTestExecution.setEnvironment(environmentDetails != null ? environmentDetails.getName() : null);
                        // Status details
                        TestExecutionStatus executionstatusDetails = null;
                        if (zephyrTestRun.getTestExecutionStatus() != null) {
                            executionstatusDetails = zephyrScaleClient.getTestExecutionStatus(token, String.valueOf(zephyrTestRun.getTestExecutionStatus().getId()));
                            if (executionstatusDetails == null) {
                                log.warn("Status details are null for test execution status ID: {}", zephyrTestRun.getTestExecutionStatus().getId());
                            }
                        } else {
                            log.warn("Test execution status is null for test case: {}", zephyrTestRun.getKey());
                        }
                        if(executionstatusDetails != null) {
                            zephyrScaleTestExecution.setTestExecutionStatusName(executionstatusDetails.getName());
                            zephyrScaleTestExecution.setTestExecutionStatusId(executionstatusDetails.getId());
                        }else{
                            zephyrScaleTestExecution.setTestExecutionStatusName(null);
                            zephyrScaleTestExecution.setTestExecutionStatusId(-1);
                        }
                        // Project details
                        zephyrScaleTestExecution.setJiraProjectId(project.getJiraProjectId());
                        zephyrScaleTestExecution.setProjectId(project.getId());
                        zephyrScaleTestExecution.setProjectKey(project.getKey());
                        zephyrScaleTestExecution.setKey(zephyrTestRun.getKey());
                        zephyrScaleTestExecution.setActualEndDate(zephyrTestRun.getActualEndDate());
                        zephyrScaleTestExecution.setExecutionTime(zephyrTestRun.getExecutionTime());
                        zephyrScaleTestExecution.setExecutedById(zephyrTestRun.getExecutedById());
                        zephyrScaleTestExecution.setAssignedToId(zephyrTestRun.getAssignedToId());
                        zephyrScaleTestExecution.setComment(zephyrTestRun.getComment());
                        zephyrScaleTestExecution.setAutomated(zephyrTestRun.isAutomated());
                        zephyrScaleTestExecution.setExecutedOn(zephyrTestRun.getExecutedOn());
                        // Test cycle details
                        TestCycle testCycleDetails = null;
                        if (zephyrTestRun.getTestCycle() != null) {
                            String testCycleId = String.valueOf(zephyrTestRun.getTestCycle().getId());
                            log.info("Fetching TestCycle details for test cycle ID: {}", testCycleId);
                            testCycleDetails = zephyrScaleClient.getTestCycle(token, testCycleId);
                            if (testCycleDetails == null) {
                                log.warn("TestCycle details are null for test cycle ID: {}", testCycleId);
                            }
                        } else {
                            log.warn("Test cycle is null for test execution: {}", zephyrTestRun.getKey());
                        }
                        if (testCycleDetails != null) {
                            zephyrScaleTestExecution.setCycleKey(testCycleDetails.getKey());
                            zephyrScaleTestExecution.setCycleName(testCycleDetails.getName());
                            if (testCycleDetails.getStatus() != null) {
                                Status statusDetails = zephyrScaleClient.getStatus(token, String.valueOf(testCycleDetails.getStatus().getId()));
                                if (statusDetails != null && statusDetails.getName() != null) {
                                    zephyrScaleTestExecution.setCycleStatusName(statusDetails.getName());
                                } else {
                                    log.warn("Status or status name is null for test cycle ID: {}", testCycleDetails.getId());
                                    zephyrScaleTestExecution.setCycleStatusName(null);
                                }
                            } else {
                                log.warn("Status is null for test cycle ID: {}", testCycleDetails.getId());
                                zephyrScaleTestExecution.setCycleStatusName(null);
                            }
                            zephyrScaleTestExecution.setCycleKey(testCycleDetails.getKey());
                            zephyrScaleTestExecution.setCycleName(testCycleDetails.getName());
                            zephyrScaleTestExecution.setCycleStatusId(String.valueOf(testCycleDetails.getStatus().getId()));
                            zephyrScaleTestExecution.setTestCycleFolder(String.valueOf(testCycleDetails.getFolder()));
                            zephyrScaleTestExecution.setDescription(testCycleDetails.getDescription());
                            zephyrScaleTestExecution.setPlannedStartDate(testCycleDetails.getPlannedStartDate());
                            zephyrScaleTestExecution.setPlannedEndDate(testCycleDetails.getPlannedEndDate());
                            zephyrScaleTestExecution.setTestCycleOwner(testCycleDetails.getOwner());
                            zephyrScaleTestExecution.setTestCycleLinks(testCycleDetails.getLinks());
                          //  zephyrScaleTestExecution.setCycleComment(.getC);
                        } else {
                            zephyrScaleTestExecution.setCycleKey(null);
                            zephyrScaleTestExecution.setCycleName(null);
                            zephyrScaleTestExecution.setCycleStatusId(null);
                           // zephyrScaleTestExecution.setCycleStatusName(null);
                            zephyrScaleTestExecution.setTestCycleFolder(null);
                            zephyrScaleTestExecution.setDescription(null);
                            zephyrScaleTestExecution.setPlannedStartDate(null);
                            zephyrScaleTestExecution.setPlannedEndDate(null);
                            zephyrScaleTestExecution.setTestCycleOwner(null);
                            zephyrScaleTestExecution.setTestCycleLinks(null);
                        }
                        return zephyrScaleTestExecution;
                    }).collect(Collectors.toList());
            List<ZephyrTestRun> testRunToSave = new ArrayList<>();
            zephyrScaleTestExecutions.forEach(testExecution -> {
                Optional<ZephyrTestRun> existingExecution = testRunService.findById(String.valueOf(testExecution.getId()));
                if (existingExecution.isPresent()) {
                    ZephyrTestRun existing = existingExecution.get();
                    existing.setKey(testExecution.getKey());
                    existing.setEnvironmentId(testExecution.getEnvironmentId());
                    existing.setTestCaseName(testExecution.getTestCaseName());
                    existing.setTestCaseId(testExecution.getTestCaseId());
                    existing.setType(testExecution.getType());
                    existing.setTestcaseKey(testExecution.getTestcaseKey());
                    existing.setEnvironment(testExecution.getEnvironment());
                    existing.setJiraProjectVersion(testExecution.getJiraProjectVersion());
                    existing.setTestExecutionStatusId(testExecution.getTestExecutionStatusId());
                    existing.setTestExecutionStatusName(testExecution.getTestExecutionStatusName());
                    existing.setActualEndDate(testExecution.getActualEndDate());
                    existing.setExecutionTime(testExecution.getExecutionTime());
                    existing.setExecutedById(testExecution.getExecutedById());
                    existing.setAssignedToId(testExecution.getAssignedToId());
                    existing.setComment(testExecution.getComment());
                    existing.setAutomated(testExecution.isAutomated());
                    existing.setTestExecutionLinks(testExecution.getTestExecutionLinks());
                    existing.setExecutedOn(testExecution.getExecutedOn());
                    existing.setOSVersion(testExecution.getOSVersion());
                    existing.setDeviceName(testExecution.getDeviceName());
                    existing.setExecutionType(testExecution.getExecutionType());
                    existing.setCycleKey(testExecution.getCycleKey());
                    existing.setCycleName(testExecution.getCycleName());
                    existing.setCycleStatusName(testExecution.getCycleStatusName());
                    existing.setCycleStatusId(testExecution.getCycleStatusId());
                    existing.setTestCycleFolder(testExecution.getTestCycleFolder());
                    existing.setDescription(testExecution.getDescription());
                    existing.setPlannedStartDate(testExecution.getPlannedStartDate());
                    existing.setPlannedEndDate(testExecution.getPlannedEndDate());
                    existing.setTestCycleOwner(testExecution.getTestCycleOwner());
                    existing.setTestCycleLinks(testExecution.getTestCycleLinks());
                    testRunToSave.add(existing);
                } else {
                    testRunToSave.add(testExecution);
                }
            });
            testRunService.saveAll(testRunToSave);
            totalSavedCount += testRunToSave.size();
            log.info("Saved {} test cases in this batch, total saved so far: {}", testRunToSave.size(), totalSavedCount);
            startAt += maxResults;
            if (testExecutionInfo.getTotal() < startAt) {
                isLast = true;
            }
            log.info("TestExecutions count from server : {}", zephyrScaleTestExecutions.size());
        } while (!isLast);
    }
}
