# Unity-Basic-Behavior-Tree
Just a basic code-based behavior tree for Unity

Very simple to use. An example:
(NOTE from Belzecue: I replaced the original example with a new one, because of how the code has been refactored to eliminate garbage generation.  Note also that I run the tick in Update, even though I'm moving rigidbody's, because I like to run manual Physics at 1/60 to match screen refresh. See here: https://docs.unity3d.com/ScriptReference/Physics.Simulate.html)
```cs
using UnityEngine;
using System;

namespace Fergicide
{
	public class BT_ChasePlayer : MonoBehaviour
	{
		public GameObject target;
		public float triggerRadius = 10;
		public float stopRadius = 2f;
		public float stopGap = 0.5f;
		public float chaseSpeed = 4;
		public float lookAtSpeed = 5;

		BehaviorTree tree;
		Transform thisTransform, targetTransform;
		Rigidbody thisRigidbody;
		Vector3 targetVector;
		float sqrDistToTarget;

		private void Awake()
		{
			thisTransform = transform;
			targetTransform = target.transform;
			thisRigidbody = GetComponent<Rigidbody>();

			//New tree
			tree = new BehaviorTree(
			  //New repeating segment, by default will run forever
			  new RepeaterNode(
				//Starts new sequence, the first time it receives a failure from it's child, it will stop
				new AndNode(
				  new RunNode(GetTarget, out onCompleteGetTarget),
				  new RunNode(FaceTarget, out onCompleteFaceTarget),
				  new OrNode(
					new RunNode(MoveAwayFromTarget, out onCompleteMoveAwayFromTarget),
					new RunNode(MoveTowardTarget, out onCompleteMoveTowardTarget)
				  )
				)
			  )
			);
		}

		void Update()
		{
			tree.Tick();
		}

		private Action<NodeResult> onCompleteGetTarget;
		private void GetTarget()
		{
			targetVector = (targetTransform.position - thisTransform.position);
			sqrDistToTarget = targetVector.sqrMagnitude;
			bool _result = sqrDistToTarget < triggerRadius * triggerRadius;

			onCompleteGetTarget( _result ? NodeResult.Succeeded : NodeResult.Failed);
		}

		private Action<NodeResult> onCompleteFaceTarget;
		private void FaceTarget()
		{
			float angleDiff = Vector3.Angle(thisTransform.forward, targetVector);
			if (angleDiff > 1)
			{
				Vector3 cross = Vector3.Cross(thisTransform.forward, targetVector);
				thisRigidbody.AddTorque(cross * angleDiff * Time.deltaTime * lookAtSpeed);
			}

			onCompleteFaceTarget(NodeResult.Succeeded);
		}

		private Action<NodeResult> onCompleteMoveTowardTarget;
		private void MoveTowardTarget()
		{
			if (
				// Hunter outside stopRadius.
				sqrDistToTarget > ((stopRadius + stopGap) * (stopRadius + stopGap))
				// And Hunter has not reached top speed.
				&& thisRigidbody.velocity.sqrMagnitude < chaseSpeed * chaseSpeed
			)
			{
				thisRigidbody.AddForce(
					targetVector.normalized *
					Time.deltaTime *
					chaseSpeed *
					500
				);
			}

			onCompleteMoveTowardTarget(NodeResult.Succeeded);
		}

		private Action<NodeResult> onCompleteMoveAwayFromTarget;
		private void MoveAwayFromTarget()
		{
			if (
				// Hunter is inside stopRadius.
				sqrDistToTarget < ((stopRadius - stopGap) * (stopRadius - stopGap))
				// And Hunter has not reached top speed.
				&& thisRigidbody.velocity.sqrMagnitude < chaseSpeed * chaseSpeed
			)
			{
				thisRigidbody.AddForce(
					-targetVector.normalized *
					Time.deltaTime *
					chaseSpeed *
					500
				);

				onCompleteMoveAwayFromTarget(NodeResult.Succeeded);
			}
			else
			{
				onCompleteMoveAwayFromTarget(NodeResult.Failed);
			}
		}
	}
}
```
This will simply keep running (via the RepeaterNode), then in sequence run a set of actions until it receives a failed result. 

Demonstration video (and confirmation of no garbage allocation) here: https://www.youtube.com/watch?v=rlfCCPIA2QI

