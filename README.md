# Untitled

1.Add in projects.js

tolunaConfig: {
surveyId: String,
quotaId: String,
waveId: String,
// panelGuid: String,
country: String
},

1. index.js

```
router.use('/tolunaapi', require('./tolunaapi'));
router.use('/toluna', require('./toluna')); 
```

[3.in](http://3.in) projects.js routes

```
router.post('/', async (req, res) => {
  try {
    const projectData = { ...req.body };
    projectData.projectType = 'single';
    
   
    // Extract Toluna-specific data
    const { tolunaConfig, ...otherData } = projectData;
    const isApiProject = projectData.isApiProject !== undefined ? projectData.isApiProject : false;
    
    console.log('Validation check - isApiProject type:', typeof projectData.isApiProject);
    console.log('Validation check - isApiProject value:', projectData.isApiProject);
    
    // Create the project with Toluna data
    const project = await Project.create({
      ...otherData,
      isApiProject: isApiProject || false,
      tolunaConfig: tolunaConfig || null
    });
    
  
    
    const links = generateLinks(project._id , isApiProject, tolunaConfig);
    Object.assign(project, links);
    
    console.log('=== BEFORE SAVE ===');
    console.log('Project before save - tolunaConfig:', project.tolunaConfig);
    
    await project.save();
    
  
    
    // Verify the data was actually saved to database
    const savedProject = await Project.findById(project._id);
   
    
    const completeProject = await Project.findById(project._id)
      .populate('clientId', 'clientName clientCode country');
    
    res.status(201).json({ success: true, data: completeProject });
  } catch (error) {
    console.error('Error creating single project:', error);
    if (error.name === 'ValidationError') {
      console.error('Validation errors:', error.errors);
    }
    res.status(400).json({ success: false, error: error.message });
  }
});
```

```jsx
router.put('/:id', async (req, res) => {
  try {
    const { isApiProject, tolunaConfig, ...otherData } = req.body;
    
    const project = await Project.findByIdAndUpdate(
      req.params.id, 
      { 
        ...otherData, 
        isApiProject: isApiProject !== undefined ? isApiProject : false,
        tolunaConfig: tolunaConfig !== undefined ? tolunaConfig : null,
        updatedAt: new Date() 
      }, 
      { new: true, runValidators: true }
    );
    
    if (!project) {
      return res.status(404).json({ success: false, error: 'Project not found' });
    }
    
    
    if (project.parentProject) {
      await updateGroupStatus(project.parentProject);
    }
    
    res.json({ success: true, data: project });
  } catch (error) {
    res.status(400).json({ success: false, error: error.message });
  }
});

```

4.toluna.js in routes

```
const express = require('express');
const router = express.Router();
const axios = require('axios');
const { Project } = require('../models/Project');

// Get panel GUID from environment variable
const TOLUNA_PANEL_GUID = process.env.TOLUNA_PANEL_GUID;

// Generate member code from zid parameter or create random one
const getMemberCode = (zid) => {
  if (zid && zid !== '[identifier]') {
    return zid; // Use the provided zid
  }
  // Generate random member code if not provided
  return `ZEPHYR_${Date.now()}_${Math.random().toString(36).substr(2, 5).toUpperCase()}`;
};

// Register member with Toluna
const registerTolunaMember = async (memberCode) => {
  try {
    const url = 'https://training.ups.toluna.com/IntegratedPanelService/api/Respondent';
    
    const memberData = {
      PartnerGUID: TOLUNA_PANEL_GUID,
      MemberCode: memberCode,
      BirthDate: "6/21/1992", // Default demo data
      PostalCode: "10001",    // Default demo data
      IsActive: true,
      IsTest: true
    };

    console.log(`Registering Toluna member: ${memberCode}`);
    
    const response = await axios.post(url, memberData, {
      headers: {
        'Accept': 'application/json;version=2.0',
        'Content-Type': 'application/json'
      },
      timeout: 30000
    });

    console.log(`Member registration status: ${response.status}`);
    return { success: true, memberCode };
  } catch (error) {
    console.error('Toluna member registration error:', error.response?.data || error.message);
    return { success: false, error: error.response?.data || error.message };
  }
};

// Add demographics to member
const updateMemberDemographics = async (memberCode) => {
  try {
    const url = 'https://training.ups.toluna.com/IntegratedPanelService/api/Respondent';
    
    const updateData = {
      PartnerGUID: TOLUNA_PANEL_GUID,
      MemberCode: memberCode,
      AnsweredQuestions: [
        {
          QuestionID: 1001007, // Gender
          AnswerID: 2000247    // Male
        },
        {
          QuestionID: 1001538, // Age
          AnswerID: 2006352    // Age group 25-34
        }
      ]
    };

    console.log(`Adding demographics to member: ${memberCode}`);

    const response = await axios.put(url, updateData, {
      headers: {
        'Accept': 'application/json;version=2.0',
        'Content-Type': 'application/json;version=2.0'
      },
      timeout: 30000
    });

    console.log(`Demographics added, status: ${response.status}`);
    return { success: true };
  } catch (error) {
    console.error('Toluna demographics error:', error.response?.data || error.message);
    return { success: false, error: error.response?.data || error.message };
  }
};

// Generate survey invite
const generateSurveyInvite = async (memberCode, quotaId) => {
  try {
    const url = `https://training.ups.toluna.com/IPExternalSamplingService/ExternalSample/${TOLUNA_PANEL_GUID}/${memberCode}/Invite/${quotaId}`;
    
    console.log(`Generating invite for quota: ${quotaId}`);
    console.log(`URL: ${url}`);
    
    const response = await axios.get(url, {
      headers: {
        'API_AUTH_KEY': process.env.TOLUNA_API_KEY,
        'Accept': 'application/json;version=2.0'
      },
      timeout: 30000
    });

    console.log(`Survey invite status: ${response.status}`);
    console.log(`Survey URL: ${response.data.URL}`);
    
    return { success: true, data: response.data };
  } catch (error) {
    console.error('Toluna invite generation error:', error.response?.data || error.message);
    return { success: false, error: error.response?.data || error.message };
  }
};

// Main survey start endpoint - ONLY FOR API PROJECTS
router.get('/survey/start', async (req, res) => {
  try {
    const { pid, test, zid } = req.query;
    
    if (!pid) {
      return res.status(400).json({ error: 'Project ID required' });
    }

    // Validate that panel GUID is set
    if (!TOLUNA_PANEL_GUID) {
      return res.status(500).json({ error: 'Toluna panel GUID not configured' });
    }

    // Get project data
    const project = await Project.findById(pid);
    
    if (!project || !project.isApiProject || !project.tolunaConfig) {
      return res.status(404).json({ 
        error: 'Project not found or not an API project or missing Toluna config' 
      });
    }

    const quotaId = project.tolunaConfig.quotaId;

    // Generate member code from zid parameter or create random one
    const memberCode = getMemberCode(zid);
    console.log(`Using member code: ${memberCode} (from zid: ${zid})`);
    
    // Step 1: Register member
    const registrationResult = await registerTolunaMember(memberCode);
    if (!registrationResult.success) {
      return res.status(500).json({ error: 'Failed to register member: ' + registrationResult.error });
    }

    // Step 2: Add demographics (optional - commented out for now)
    // const demographicsResult = await updateMemberDemographics(memberCode);
    // if (!demographicsResult.success) {
    //   return res.status(500).json({ error: 'Failed to add demographics: ' + demographicsResult.error });
    // }

    // Step 3: Generate survey invite
    const inviteResult = await generateSurveyInvite(memberCode, quotaId);
    if (!inviteResult.success) {
      return res.status(500).json({ error: 'Failed to generate invite: ' + inviteResult.error });
    }

    // Update project metrics
    await Project.findByIdAndUpdate(pid, {
      $inc: { 'metrics.totalStarts': 1 }
    });

    // Redirect to Toluna survey
    res.redirect(inviteResult.data.URL);

  } catch (error) {
    console.error('Toluna survey start error:', error);
    res.status(500).json({ error: 'Failed to start Toluna survey: ' + error.message });
  }
});

module.exports = router;
```

[5.tolunaapi.](http://5.tolunaapi.sj)js in routes

```
const express = require('express');
const router = express.Router();
const axios = require('axios');
const Client = require('../models/Client');
const tolunaReferenceService = require('../services/tolunaReferenceService');
const tolunaQAService = require('../services/tolunaQAService');
const extractSurveyUrls = (survey) => {
    // Try to find actual survey URLs from Toluna response
    const liveUrl = survey.SurveyURL || survey.LiveURL || survey.URL || null;
    const testUrl = survey.TestURL || survey.PreviewURL || (liveUrl ? liveUrl + '&preview=true' : null);
    
    return { liveUrl, testUrl };
  };
  
router.get('/surveys', async (req, res) => {
  try {
    const { clientId, country, pageSize = 10, page = 1 } = req.query;
 
    if (!clientId || !country) {
      return res.status(400).json({ error: 'Client and Country are required parameters' });
    }
 
    const client = await Client.findById(clientId);
    if (!client) {
      return res.status(404).json({ error: 'Client not found' });
    }
 
    const credentialsByCountry = {
      'United States': {
        panelGUID: process.env.TOLUNA_US_PANEL_GUID,
        apiAuthKey: process.env.TOLUNA_API_KEY
      },
      'United Kingdom': {
        panelGUID: process.env.TOLUNA_UK_PANEL_GUID,
        apiAuthKey: process.env.TOLUNA_API_KEY
      }
    };
 
    const creds = credentialsByCountry[country];
    if (!creds || !creds.panelGUID || !creds.apiAuthKey) {
      return res.status(400).json({
        error: 'Toluna credentials not configured for this country'
      });
    }
 
    const ipEsUrl = process.env.TOLUNA_IP_ES_URL;
    if (!ipEsUrl) {
      return res.status(500).json({ error: 'Toluna service URL not configured' });
    }
 
    const tolunaUrl = `${ipEsUrl}/ExternalSample/${creds.panelGUID}/Quotas?includeRoutables=false`;
 
    
    const response = await axios.get(tolunaUrl, {
      headers: { 'API_AUTH_KEY': creds.apiAuthKey },
      timeout: 30000
    });
 
    let surveys = response.data?.Surveys || [];
    
    
    const enhancedSurveys = await Promise.all(
        surveys.map(async (survey) => {
          try {
            const currencyName = survey.Price?.CurrencyID
              ? await tolunaReferenceService.getCurrencyName(survey.Price.CurrencyID)
              : 'Unknown';
            
            const deviceTypeNames = survey.DeviceTypeIDs && survey.DeviceTypeIDs.length
              ? await tolunaReferenceService.getDeviceTypeNames(survey.DeviceTypeIDs)
              : 'Unknown';
            
            // Extract survey URLs
            const { liveUrl, testUrl } = extractSurveyUrls(survey);
            
            return {
              ...survey,
              currencyName,              
              deviceTypeNames,
              extractedUrls: { liveUrl, testUrl } // Add this

            };
          } catch (error) {
            console.error('Error enhancing survey data:', error);
            return {
              ...survey,
              currencyName: 'Unknown',
              deviceTypeNames: 'Unknown',
              extractedUrls: { liveUrl: null, testUrl: null }
            };
          }
        })
      );
      
    
    const startIndex = (page - 1) * pageSize;
    const endIndex = startIndex + parseInt(pageSize, 10);
    const paginatedSurveys = enhancedSurveys.slice(startIndex, endIndex);
 
    res.json({
      surveys: paginatedSurveys,
      totalCount: enhancedSurveys.length,
      page: parseInt(page, 10),
      pageSize: parseInt(pageSize, 10),
      totalPages: Math.ceil(enhancedSurveys.length / pageSize)
    });
 
  } catch (error) {
    console.error('Error fetching surveys from Toluna:', error.message);
    
    if (error.response) {
      return res.status(error.response.status || 502).json({
        error: 'Failed to fetch surveys from Toluna',
        details: error.response.data
      });
    }
    
    res.status(500).json({ error: 'Internal server error' });
  }
});
 
 
router.post('/refresh-reference-data', async (req, res) => {
  try {
    
    await tolunaReferenceService.getCurrencies(false);
    await tolunaReferenceService.getDeviceTypes(false);
    
    res.json({ success: true, message: 'Reference data cache refreshed' });
  } catch (error) {
    console.error('Error refreshing reference data:', error);
    res.status(500).json({ error: 'Failed to refresh reference data' });
  }
});
 
 
router.post('/targeting-info', async (req, res) => {
  try {
    const { quotas, country } = req.body;
 
    if (!quotas || !country) {
      return res.status(400).json({ error: 'Quotas and country are required' });
    }
 
    
    const countryToCultureId = {
      'United States': 1,
      'United Kingdom': 5
    };
 
    const cultureId = countryToCultureId[country] || 1;
 
    let qaData = [];
    try {
      
      qaData = await tolunaQAService.getQuestionsAndAnswers([cultureId]);
      console.log(`Successfully retrieved ${qaData.length} questions from Q&A service`);
    } catch (error) {
      console.error('Failed to fetch Q&A data, using fallbacks only:', error.message);
      
    }
 
    
    const targetingSet = new Set();
    
    
    quotas.forEach(quota => {
      if (quota.Layers && quota.Layers.length) {
        quota.Layers.forEach(layer => {
          if (layer.SubQuotas && layer.SubQuotas.length) {
            layer.SubQuotas.forEach(subQuota => {
              if (subQuota.QuestionsAndAnswers && subQuota.QuestionsAndAnswers.length) {
                subQuota.QuestionsAndAnswers.forEach(qa => {
                  const questionText = tolunaQAService.getQuestionText(qaData, qa.QuestionID);
                  
                  const answerTexts = qa.AnswerIDs.map(answerID =>
                    tolunaQAService.getAnswerText(qaData, qa.QuestionID, answerID)
                  );
                  
                  
                  const targetingKey = `${questionText}:${answerTexts.sort().join(',')}`;
                  
                  
                  if (!targetingSet.has(targetingKey)) {
                    targetingSet.add(targetingKey);
                  }
                });
              }
            });
          }
        });
      }
    });
 
    
    const targetingInfo = Array.from(targetingSet).map(item => {
      const [question, answers] = item.split(':');
      return `${question}: ${answers}`;
    });
 
    
    if (targetingInfo.length === 0) {
      targetingInfo.push('No targeting information available');
    }
 
    res.json({ targetingInfo });
  } catch (error) {
    console.error('Error getting targeting information:', error);
    res.status(500).json({
      error: 'Failed to get targeting information',
      details: process.env.NODE_ENV === 'development' ? error.message : undefined
    });
  }
});
 
// Get study types
router.get('/study-types', async (req, res) => {
  try {
    const studyTypes = await tolunaReferenceService.getStudyTypes();
    res.json({ success: true, data: studyTypes });
  } catch (error) {
    console.error('Error fetching study types:', error);
    res.status(500).json({
      success: false,
      error: 'Failed to fetch study types',
      details: process.env.NODE_ENV === 'development' ? error.message : undefined
    });
  }
});
 
// Get question categories
router.get('/question-categories', async (req, res) => {
  try {
    const questionCategories = await tolunaReferenceService.getQuestionCategories();
    res.json({ success: true, data: questionCategories });
  } catch (error) {
    console.error('Error fetching question categories:', error);
    res.status(500).json({
      success: false,
      error: 'Failed to fetch question categories',
      details: process.env.NODE_ENV === 'development' ? error.message : undefined
    });
  }
});
 
// Get question data
router.get('/question-data/:questionID/:cultureID', async (req, res) => {
  try {
    const { questionID, cultureID } = req.params;
    const questionData = await tolunaReferenceService.getQuestionData(parseInt(questionID), parseInt(cultureID));
    res.json({ success: true, data: questionData });
  } catch (error) {
    console.error('Error fetching question data:', error);
    res.status(500).json({
      success: false,
      error: 'Failed to fetch question data',
      details: process.env.NODE_ENV === 'development' ? error.message : undefined
    });
  }
});
 
router.post('/enhanced-targeting-info', async (req, res) => {
  try {
    const { quotas, country, survey } = req.body; // Added survey parameter
 
    if (!quotas || !country) {
      return res.status(400).json({ error: 'Quotas and country are required' });
    }
 
    const countryToCultureId = {
      'United States': 1,
      'United Kingdom': 5
    };
 
    const cultureId = countryToCultureId[country] || 1;
 
    let qaData = [];
    try {
      qaData = await tolunaQAService.getQuestionsAndAnswers([cultureId]);
    } catch (error) {
      console.error('Failed to fetch Q&A data:', error.message);
    }
 
    // Get study types and question categories
    let allStudyTypes = [];
    let allQuestionCategories = [];
    
    try {
      allStudyTypes = await tolunaReferenceService.getStudyTypes();
    } catch (error) {
      console.error('Failed to fetch study types:', error);
    }
    
    try {
      allQuestionCategories = await tolunaReferenceService.getQuestionCategories();
    } catch (error) {
      console.error('Failed to fetch question categories:', error);
    }
 
    const targetingInfo = [];
    const questionMap = new Map();
    const usedStudyTypeIDs = new Set();
    const usedCategoryIDs = new Set();
    
    quotas.forEach(quota => {
      if (quota.Layers && quota.Layers.length) {
        quota.Layers.forEach(layer => {
          if (layer.SubQuotas && layer.SubQuotas.length) {
            layer.SubQuotas.forEach(subQuota => {
              if (subQuota.QuestionsAndAnswers && subQuota.QuestionsAndAnswers.length) {
                subQuota.QuestionsAndAnswers.forEach(qa => {
                  const questionText = tolunaQAService.getQuestionText(qaData, qa.QuestionID);
                  
                  const answerTexts = qa.AnswerIDs.map(answerID =>
                    tolunaQAService.getAnswerText(qaData, qa.QuestionID, answerID)
                  );
                  
                  // Group by question
                  if (!questionMap.has(qa.QuestionID)) {
                    questionMap.set(qa.QuestionID, {
                      questionID: qa.QuestionID,
                      questionText,
                      answers: new Set()
                    });
                  }
                  
                  answerTexts.forEach(answer => {
                    questionMap.get(qa.QuestionID).answers.add(answer);
                  });
                });
              }
            });
          }
        });
      }
    });
 
    // Convert map to array
    const questions = Array.from(questionMap.values()).map(q => ({
      questionID: q.questionID,
      questionText: q.questionText,
      answers: Array.from(q.answers)
    }));
 
    
    let surveyStudyType = null;
    if (survey && survey.StudyTypeID) {
      surveyStudyType = allStudyTypes.find(st => st.StudyTypeID === survey.StudyTypeID);
    }
 
    res.json({
      targetingInfo: questions,
      surveyStudyType,
      
    });
  } catch (error) {
    console.error('Error getting enhanced targeting information:', error);
    res.status(500).json({
      error: 'Failed to get targeting information',
      details: process.env.NODE_ENV === 'development' ? error.message : undefined
    });
  }
});
// Add this to your existing tolunaapi.js
module.exports = router;
```

1. surveyUrlService.js

```
const axios = require('axios');

const SURVEY_URL_SERVICE_BASE = process.env.SURVEY_URL_SERVICE_BASE || 'http://localhost:5001';

class SurveyUrlService {
  static async generateSurveyUrl(surveyData) {
    try {
      const { country, surveyId, surveyName, quotaId, demographics = {} } = surveyData;
      
      const response = await axios.post(
        `${SURVEY_URL_SERVICE_BASE}/api/survey-link/generate/${country}/${quotaId}`,
        {
          surveyId,
          surveyName,
          demographics
        }
      );

      if (response.data.success) {
        return {
          success: true,
          liveUrl: response.data.surveyUrl,
          testUrl: response.data.surveyUrl + '&test=true',
          linkId: response.data.linkId
        };
      }
      throw new Error(response.data.message || 'Failed to generate survey URL');
    } catch (error) {
      console.error('Survey URL generation error:', error);
      return {
        success: false,
        error: error.message || 'Survey URL service unavailable'
      };
    }
  }

  static async getSurveyAnalytics(linkId) {
    try {
      const response = await axios.get(
        `${SURVEY_URL_SERVICE_BASE}/api/survey-link/analytics/${linkId}`
      );
      return response.data;
    } catch (error) {
      console.error('Survey analytics error:', error);
      return { success: false, error: error.message };
    }
  }
}

module.exports = SurveyUrlService;
```

7.helpers.js

```
const generateLinks = (projectId, isApiProject = false, tolunaConfig = null) => {
  const baseUrl = process.env.BASE_URL || 'http://localhost:5001';
  const surveyBaseUrl = process.env.SURVEY_BASE_URL || 'https://survey.example.com';
  
  
  const liveToken = crypto.randomBytes(16).toString('hex');
  const testToken = crypto.randomBytes(16).toString('hex');
  if (isApiProject && tolunaConfig) {
    return {
      surveyLinks: {
        linkType: 'single',
        liveLink: `${baseUrl}/api/toluna/survey/start?pid=${projectId}&zid=[identifier]`,
        testLink: `${baseUrl}/api/toluna/survey/start?pid=${projectId}&test=true&zid=[identifier]`,
        variableMapping: true,
        generatedAt: new Date(),
        generatedBy: 'toluna-api',
        isApiProject: true
      },
      projectLinks: {
        live: `${baseUrl}/api/toluna/survey/start?pid=${projectId}&zid=[identifier]`,
        test: `${baseUrl}/api/toluna/survey/start?pid=${projectId}&test=true&zid=[identifier]`
      }
    };
  }
  else {
    return {
      surveyLinks: {
        linkType: 'single',
        liveLink: '', 
        testLink: '', 
        variableMapping: true
      },
      
      projectLinks: {
        live: `${baseUrl}/api/projects/${projectId}/survey?token=${liveToken}`,
        test: `${baseUrl}/api/projects/${projectId}/survey?token=${testToken}&test=1`
      },
      
    
      // redirectLinks: {
      //   completeStatus: `http://localhost:5001/Campaign?transaction_end=10&zid=[identifier]&pid=${projectId}`,
      //   terminateStatus: `http://localhost:5001/Campaign?transaction_end=20&zid=[identifier]&pid=${projectId}`,
      //   overQuotaStatus: `http://localhost:5001/Campaign?transaction_end=40&zid=[identifier]&pid=${projectId}`,
      //   qualityTermStatus: `http://localhost:5001/Campaign?transaction_end=30&zid=[identifier]&pid=${projectId}`,
      //   surveyCloseStatus: `http://localhost:5001/Campaign?transaction_end=70&zid=[identifier]&pid=${projectId}`
      // },
      redirectLinks: {
        completeStatus: `http://localhost:5001/Campaign?transaction_end=10&pid=${projectId}&zid=[identifier]`,
        terminateStatus: `http://localhost:5001/Campaign?transaction_end=20&pid=${projectId}&zid=[identifier]`,
        overQuotaStatus: `http://localhost:5001/Campaign?transaction_end=40&pid=${projectId}&zid=[identifier]`,
        qualityTermStatus: `http://localhost:5001/Campaign?transaction_end=30&pid=${projectId}&zid=[identifier]`,
        surveyCloseStatus: `http://localhost:5001/Campaign?transaction_end=70&pid=${projectId}&zid=[identifier]`
      },
      
      postbackLinks: {
        postbackURL: `${baseUrl}/api/callback/success?projectId=${projectId}`,
        postbackActiveURL: `${baseUrl}/api/callback/active?projectId=${projectId}`
      }
    };
  }
 
};

```

[8.in](http://8.in) single project

```
  const loadProjectForEdit = async (projectId) => {
    setLoadingProject(true);
    // console.log('📖 Loading project data for editing:', projectId);
    
    try {
      const result = await apiCall(projectAPI.getProject, projectId);
      
      if (result.success) {
        const project = result.data.data;
        // console.log('📖 Project data loaded:', project);
        
        
        setFormData({
          projectType: 'single',
          client: project.clientId?._id || project.client || '',
          projectName: project.projectName || '',
          projectManager: project.projectManager || '',
          country: project.country || '',
          language: project.language || '',
          projectCategory: project.projectCategory || '',
          description: project.description || '',
          loi: project.loi || project.LOI || '',
          ir: project.ir || project.IR || '',
          sampleSize: project.sampleSize || 0,
          clickQuota: project.clickQuota || 0,
          currency: project.currency || 'USD',
          projectCPI: project.projectCPI || '',
          supplierCPI: project.supplierCPI || '',
          startDate: project.startDate ? new Date(project.startDate).toISOString().split('T')[0] : '',
          endDate: project.endDate ? new Date(project.endDate).toISOString().split('T')[0] : '',
          filters: {
            preScreen: project.filters?.preScreen || false,
            exclude: project.filters?.exclude || false,
            tSign: project.filters?.tSign || false,
            geoLocation: project.filters?.geoLocation || false,
            captcha: project.filters?.captcha || false,
            reInvite: project.filters?.reInvite || false,
            uniqueIP: project.filters?.uniqueIP || false,
            pii: project.filters?.pii || false,
            reRoute: project.filters?.reRoute || false,
            speeder: project.filters?.speeder || false,
            centryLOI: project.filters?.centryLOI || false, // Added centryLOI filter
            dynamicThanksUrl: project.filters?.dynamicThanksUrl || false
          },
          deviceFilter: {
            mobile: project.deviceFilter?.mobile !== undefined ? project.deviceFilter.mobile : true,
            tablet: project.deviceFilter?.tablet !== undefined ? project.deviceFilter.tablet : true,
            desktop: project.deviceFilter?.desktop !== undefined ? project.deviceFilter.desktop : true
          },
          notes: project.notes || '',
          isApiProject: project.isApiProject || false,
        tolunaConfig: project.tolunaConfig || null,
        status: project.status || 'rfq'
        });
        
        setIsEditMode(true);
        // console.log('✅ Form populated with project data');
      } else {
       toast.error('Failed to load project: ' + result.error.message);
        
        if (onNavigate) {
          onNavigate('project-list');
        }
      }
    } catch (error) {
      toast.error('Error loading project data');
      if (onNavigate) {
        onNavigate('project-list');
      }
    }
    
    setLoadingProject(false);
  };
```

1. in apisurveyselector

```
  const handleLaunch = (survey) => {
    const selectedClientObj = clients.find(client => client._id === selectedClient);
    
    const projectData = {
      projectName: survey.SurveyName,
      projectManager: 'ADMIN',
      country: selectedCountry,
      language: 'English',
      loi: survey.LOI,
      ir: survey.IR,
      currency: survey.currencyName || 'USD',
      projectCPI: survey.Price?.Amount || '',
      sampleSize: survey.CompletesRequired || 0,
      client: selectedClient,
      clientName: selectedClientObj ? selectedClientObj.clientName : '',
      isApiProject: true,
      status:"active",
      tolunaConfig: {
        surveyId: survey.SurveyID,
        quotaId: survey.Quotas?.[0]?.QuotaID,
        waveId: survey.WaveID,
        panelGuid: selectedCountry === 'United States' ? 
          '30B0C277-BC02-49E7-AD58-A2F655DD6CC3' : '8CE18948-7454-4781-A5FF-49C954CF2DA4',
        country: selectedCountry
      }
   
  };
 
    
    navigate('/project/singleproject', {
      state: {
        prefilledData: projectData,
        source: 'toluna'
      }
    });
  };
```

1. in project detail survey link tab

```
  useEffect(() => {
      if (project.isApiProject && project.tolunaConfig && !surveyLinks.liveLink) {
          const projectZid = `${project._id}_${project.tolunaConfig.quotaId}`;
          
          
          const liveUrl = `http://localhost:5001/api/toluna/survey/start?pid=${project._id}&zid=[identifier]`;
          const testUrl = `${liveUrl}&test=true`;
          console.log('Auto-generated survey links:', { liveUrl, testUrl });
          setSurveyLinks({
              liveLink: liveUrl,
              testLink: testUrl
          });
      }
  }, [project]);
```

add TOLUNA_PANEL_GUID in env
