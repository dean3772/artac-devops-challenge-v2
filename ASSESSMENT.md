Finding 1: scikit-learn runtime version does not match the serialized ML model

What I found:
The Docker image built successfully and the FastAPI service started. /health, /ready, and /docs were reachable and returned 200 OK. However, /predict initially returned 500 Internal Server Error even with a valid request body:
{
  "text": "This movie was fantastic!"
}

The Docker logs showed that the model file was loaded, but scikit-learn emitted InconsistentVersionWarning messages. The model was serialized with scikit-learn 1.8.0, while the container installed scikit-learn 1.6.1 from requirements.txt. The actual failure happened during predict_proba:
AttributeError: 'LogisticRegression' object has no attribute 'multi_class'

Classification:
Bug.

Contractor's reasoning:
DECISIONS.md mentions that scikit-learn was pinned to 1.6.1 for stability, because newer versions may introduce breaking API changes. I disagree with this decision in the current codebase, because the provided model was already generated with scikit-learn 1.8.0. Pinning an older runtime version made the application start successfully but fail during actual prediction.

What I did:
I updated requirements.txt to use the scikit-learn version compatible with the provided model, then rebuilt and reran the Docker image. I did not modify app/ or models/, because the assignment says to treat the application as a black box. After rebuilding, the model loaded without version warnings and /health, /ready, and /predict all returned 200 OK.

Why:
For pickled ML models, the runtime library version should match the version used when the model was serialized. Fixing the dependency pin is safer and cleaner than changing application code or regenerating the model.
--
