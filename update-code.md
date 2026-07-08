const fs = require("fs");
const filePath = "/home/node/.n8n-files/state.json";
loopInfo ??= {
  currentCategoryIndex: 0,       // Which category are we on?
  currentChapterIndex: 0,        // Which chapter inside the category?
  currentTopicIndex: 0,          // Which topic inside the chapter?
  currentDifficultyIndex: 0,     // Which difficulty (easy/moderate/hard)?
  currentQuestionTypeIndex: 0,   // Which question type (Single/Multiple/Numerical)?
  currentWorkingBatchSize: 0     // Number of questions in the current batch
}